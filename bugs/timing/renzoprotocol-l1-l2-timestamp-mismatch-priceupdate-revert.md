# L1→L2 Price Update Reverts Due to Cross-Chain Timestamp Mismatch

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2024-04-renzo-findings/issues/502)
* **Affected Contracts**:

  * [`L1/xRenzoBridge.sol`](https://github.com/code-423n4/2024-04-renzo/blob/main/contracts/Bridge/L1/xRenzoBridge.sol)
  * [`L2/xRenzoDeposit.sol`](https://github.com/code-423n4/2024-04-renzo/blob/main/contracts/Bridge/L2/xRenzoDeposit.sol)
* **Vulnerability Type**: Cross-Chain Input Validation / Logic Error / Message Processing Failure

## Summary

Renzo's cross-chain price feed sends **L1 timestamps** along with price data to L2. On L2, the contract compares the received `_timestamp` against the **L2 block's timestamp**:

```solidity
if (_timestamp > block.timestamp) revert InvalidTimestamp(_timestamp);
```

However, **L1 and L2 timestamps are not synchronized** on rollup chains like Arbitrum or OP-stack (e.g., Blast). L2 timestamps are driven by **sequencer clocks**, which can drift **hours behind or ahead** of L1 timestamps within allowed bounds.

Because of this fundamental mismatch, a valid L1 timestamp can easily appear "in the future" relative to L2's timestamp. When this happens, `_updatePrice()` **reverts**, freezing legitimate price updates and potentially blocking downstream logic that relies on fresh pricing.

This makes the future-timestamp validation **invalid and unsafe** across chains.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **L1 sends price update**

   * Fetches `exchangeRate = rateProvider.getRate()`
   * Encodes `(price, block.timestamp)`
   * Sends it to L2 via CCIP or Connext

   ```solidity
   bytes memory data = abi.encode(exchangeRate, block.timestamp);
   ```

2. **L2 receives update**
   `_updatePrice(price, timestamp)` enforces invariants:

   * Price must not diverge too much
   * Timestamp must be newer than last update
   * Timestamp must **not** be "in the future"

   ```solidity
   if (_timestamp > block.timestamp) revert;
   ```

### What Actually Happens (Bug)

* On L1:
  `block.timestamp = 1716757859`

* On L2 (e.g., Blast, Arbitrum, Base):
  `block.timestamp = 1707500043`
  (This is normal; sequencer time can lag massively.)

When the price update arrives:

```solidity
if (1716757859 > 1707500043) revert;  // valid update rejected
```

Because L1 and L2 timestamps are **unrelated**, comparing them introduces **false reverts**.

### Why This Happens

Rollups **do not inherit L1 timestamps** directly:

* **Arbitrum**:

  * Sequencer timestamps may drift up to **+1 hour ahead** or **-24 hours behind**
  * Force-included messages inherit the delayed inbox timestamp or previous L2 timestamp

* **OP-stack chains** (Blast, Base, Optimism):

  * L2 timestamp follows sequencer's local clock
  * Can be significantly behind L1
  * Not guaranteed to track L1 time

Therefore, a valid L1 timestamp can easily fail the L2 check.

### Impact

* L2 rejects otherwise valid price updates
* `lastPrice` and `lastPriceTimestamp` become stale
* Protocol logic depending on updated price gets stuck or behaves incorrectly
* If multiple updates revert back-to-back, **pricing effectively freezes**
* Cross-chain dataflow becomes unreliable due to unfixable time drift

This vulnerability is not exploitable for direct theft, but causes **operational failures**, protocol freeze, and broken safety guarantees.

## Concrete Walkthrough (Alice & L2 Sequencer)

*Time: 12:00 UTC*

* **L1**

  * Timestamp = `1,000,000`
  * Alice triggers price update: Renzo sends `(price, 1,000,000)`

* **L2 sequencer** (e.g., Blast)

  * Current timestamp = `995,000` (5 minutes behind L1)

L2 receives update and runs:

```solidity
if (1_000_000 > 995_000) revert;
```

→ update fails, **even though Alice's price is valid**

The sequencer eventually moves its clock forward, but **any message that arrives within the drift window simply fails**.

If the sequencer lags by hours (allowed on Arbitrum), every update may revert until catch-up.

## Vulnerable Code Reference

### L1: Timestamp included in message

```solidity
// xRenzoBridge::sendPrice
uint256 exchangeRate = rateProvider.getRate();
bytes memory _callData = abi.encode(exchangeRate, block.timestamp);
```

### L2: Rejects valid timestamps by comparing to L2 time

```solidity
// xRenzoDeposit::_updatePrice
if (_timestamp > block.timestamp) {
    revert InvalidTimestamp(_timestamp);
}
```

### L2: Assumes monotonic L2 time across chains (false assumption)

```solidity
if (_timestamp <= lastPriceTimestamp) revert;
```

This is fine.
The bad check is the future-timestamp comparison.

## Recommended Mitigation

### 1. Remove the L2 future-timestamp check (Primary Fix)

```solidity
// REMOVE this invalid check
// if (_timestamp > block.timestamp) revert InvalidTimestamp(_timestamp);
```

This is the correct fix because L2 timestamps cannot be reliably compared to L1 timestamps.

### 2. Rely only on internal monotonicity

```solidity
require(_timestamp > lastPriceTimestamp, "stale update");
```

This ensures prices move forward without cross-chain timestamp assumptions.

### 3. Use sequence numbers instead of timestamps (Optional but robust)

L1 sends:

```solidity
(price, sequenceId)
```

L2 enforces:

```solidity
require(sequenceId > lastSequenceId);
```

This avoids time comparisons entirely.

### 4. (Optional) Signed oracle messages

Have the L1 oracle sign `(price, sequenceId)` → L2 verifies signature and monotonicity.

## Pattern Recognition Notes

* **Cross-Chain Timestamp Mismatch**
  Never compare `block.timestamp` across chains. L2 timestamps are sequencer-driven, not L1-synchronized.

* **Sequencer Drift**
  L2 timestamps can be hours behind or ahead. Rollup design explicitly allows drift to handle batching delays.

* **False Reverts = Hidden DoS**
  Safety checks that use off-chain or cross-chain assumptions can create systemic rejection of valid messages.

* **Prefer Monotonic Sequence IDs**
  Sequence numbers avoid relying on clock alignment and are deterministic across chains.

* **Ingress Validation Needs Chain-Aware Rules**
  Inputs arriving from another chain should be validated only using **internal protocol invariants**, not external chain state.

### Quick Recall (TL;DR)

* **Bug**: L2 rejects valid L1 timestamps because L1 and L2 clocks are not linked.
* **Impact**: Price updates can revert unpredictably → stale prices → protocol malfunction.
* **Fix**: Remove cross-chain timestamp comparison; rely on monotonic timestamp or sequence numbers.
