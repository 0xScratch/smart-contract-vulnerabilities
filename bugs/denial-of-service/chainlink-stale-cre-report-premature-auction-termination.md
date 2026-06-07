# Stale CRE Report Can Prematurely Terminate a New Auction Instance

* **Severity**: Medium
* **Source**: [Code4rena](https://code4rena.com/audits/2026-03-chainlink-payment-abstraction-v2/submissions/S-1103)
* **Affected Contracts**: [WorkflowRouter.sol](https://github.com/code-423n4/2026-03-chainlink/blob/5317782e3855d9547412ab5a490257d7c5e51fac/src/WorkflowRouter.sol#L86-L118), [BaseAuction.sol](https://github.com/code-423n4/2026-03-chainlink/blob/5317782e3855d9547412ab5a490257d7c5e51fac/src/BaseAuction.sol#L305-L357)
* **Vulnerability Type**: Stale State Execution / Denial of Service (DoS) / Workflow Freshness Validation

## Summary

The auction system relies on Chainlink CRE workflows to generate and submit `performUpkeep()` reports that start and end auctions. However, `WorkflowRouter.onReport()` validates only workflow authorization and target allowlisting before forwarding reports. It does **not** track report freshness, sequence numbers, timestamps, or workflow execution epochs.

Because Chainlink CRE explicitly allows delayed delivery and retry of previously failed reports, an old but still authenticated report can be delivered after protocol state has changed.

The problem becomes exploitable because `BaseAuction.performUpkeep()` identifies auctions only by **asset address**, not by a unique auction instance. As a result, a stale report originally intended to end Auction #1 can later be applied to a completely different Auction #2 for the same asset.

This allows an outdated report to prematurely terminate a healthy auction, return auction inventory, revoke solver permissions, and reset auction progress.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The system operates roughly as follows:

1. Auction #1 starts for WETH.
2. Time passes until Auction #1 reaches its end condition.
3. A CRE workflow observes this and generates:

```solidity
endedAuctions = [WETH];
```

4. The report is forwarded through `WorkflowRouter`.
5. `performUpkeep()` ends Auction #1.

Everything works correctly because the report matches the auction state that originally produced it.

### What Actually Happens (Bug)

The report contains only:

```solidity
endedAuctions = [WETH];
```

It does **not** contain:

* Auction ID
* Auction start timestamp
* Auction epoch
* Auction generation number

As a result, the report identifies only the asset, not the specific auction instance.

Suppose:

1. Auction #1 expires.
2. CRE generates an "end WETH auction" report.
3. The report is delayed or retried later.
4. Auction #1 gets ended through another execution path.
5. A brand-new Auction #2 starts for WETH.
6. The stale report finally arrives.

`WorkflowRouter` sees a valid workflow and forwards the report.

`performUpkeep()` checks only:

```solidity
s_auctionStarts[WETH] != 0
```

which simply means:

> "Is there currently some live WETH auction?"

Since Auction #2 exists, the stale report successfully executes and ends Auction #2 even though the workflow originally observed Auction #1.

The report remains authenticated but no longer matches current protocol state.

### Why This Matters

The impact extends beyond simple bookkeeping.

When the stale report terminates the new auction:

* Auction inventory is returned.
* Solver participation is interrupted.
* CowSwap relayer allowance is revoked.
* Auction progress is lost.
* Dutch auction price discovery resets.

This is especially important because Dutch auctions derive value from elapsed time.

For example:

```text
Day 0 -> $100
Day 1 -> $90
Day 2 -> $80
Day 3 -> $70
Day 4 -> $60
Day 5 -> $50
```

If market participants are waiting for the auction to reach a profitable level near Day 5, a stale report can terminate the auction just before that point.

A replacement auction begins again at:

```text
Day 0 -> $100
```

forcing participants to wait through the entire price curve again.

Thus, the impact is not merely a temporary interruption but also the loss of accumulated auction progress.

### Concrete Walkthrough (Auction #1 and Auction #2)

#### Step 1: Auction #1 Starts

```text
Asset: WETH
Auction: #1
Start: 10:00
End: 11:00
```

The auction is live.

#### Step 2: CRE Observes Auction End

At 11:00, the workflow detects that the auction should end and generates:

```solidity
endedAuctions = [WETH];
```

However, the report is delayed or fails its first delivery.

#### Step 3: Auction #1 Ends Normally

An alternative execution path ends Auction #1.

```text
Auction #1 -> Ended
```

The stale report still exists but has not yet executed.

#### Step 4: Auction #2 Starts

A fresh auction is created:

```text
Asset: WETH
Auction: #2
Start: 12:00
```

This auction has full inventory and a newly initialized price curve.

#### Step 5: Stale Report Arrives

The delayed report containing:

```solidity
endedAuctions = [WETH];
```

is submitted through `WorkflowRouter`.

The router verifies:

* Workflow allowlisted
* Target allowlisted
* Selector allowlisted

and forwards the call.

No freshness checks exist.

#### Step 6: Auction #2 Is Incorrectly Ended

Inside `performUpkeep()`:

```solidity
if (s_auctionStarts[asset] == 0) {
    revert InvalidAuction(asset);
}
```

Since Auction #2 is live:

```text
s_auctionStarts[WETH] != 0
```

the contract assumes the report is valid and ends the auction.

Result:

```text
Auction #2 terminated
Inventory returned
Allowance revoked
Price curve reset
```

even though the report was generated for Auction #1.

> **Analogy:** Imagine a manager receives a message saying "close Conference Room A." The message is delayed. The original meeting ends, and a completely different meeting starts in the same room. Hours later the delayed message arrives. The manager checks only whether Room A is currently occupied and shuts down the new meeting. The message was authentic, but it referred to the wrong meeting instance.

## Vulnerable Code Reference

### 1) WorkflowRouter Does Not Validate Report Freshness

```solidity
function onReport(
  bytes calldata metadata,
  bytes calldata report
) external whenNotPaused onlyRole(Roles.FORWARDER_ROLE) {

  bytes32 workflowId = bytes32(metadata[:32]);

  ...

  (address target, bytes memory data) = abi.decode(report, (address, bytes));

  ...

  _call(target, data);
}
```

The router verifies authorization but does not validate:

```text
Report timestamp
Report nonce
Sequence number
Workflow epoch
Freshness marker
```

Therefore, delayed or retried reports remain executable.

### 2) Auction End Logic Only Checks Asset Presence

```solidity
for (uint256 i; i < endedAuctions.length; ++i) {
    address asset = endedAuctions[i];

    if (s_auctionStarts[asset] == 0) {
        revert InvalidAuction(asset);
    }

    _onAuctionEnd(asset, hasFeeAggregator);
    delete s_auctionStarts[asset];
}
```

The code validates only:

```solidity
s_auctionStarts[asset] != 0
```

which proves only that some auction exists for the asset.

It does not verify that the auction is the same one observed by the workflow.

## Recommended Mitigation

### 1. Bind Reports To Specific Auction Instances (Primary Fix)

Include a unique auction identifier in the report payload:

```solidity
struct AuctionEndRequest {
    address asset;
    uint256 auctionStart;
}
```

or:

```solidity
struct AuctionEndRequest {
    address asset;
    uint256 auctionEpoch;
}
```

Before ending an auction:

```solidity
require(
    request.auctionEpoch == currentAuctionEpoch,
    "stale report"
);
```

This ensures reports cannot affect newly created auctions.

### 2. Add Router-Level Freshness Validation

Track report identifiers, timestamps, or sequence numbers:

```solidity
mapping(bytes32 => bool) processedReports;
```

or:

```solidity
require(
    reportTimestamp >= minimumAcceptedTimestamp,
    "stale report"
);
```

This prevents delayed reports from executing after state changes.

### 3. Reject Out-of-Order Workflow Executions

Require monotonic workflow sequence numbers:

```solidity
require(
    reportSequence > lastProcessedSequence,
    "old report"
);
```

This guarantees reports execute in order.

### 4. Add Regression Tests

Include tests covering:

* Delayed report delivery.
* Retried failed reports.
* Auction replacement before report execution.
* Multiple auctions for the same asset.
* Out-of-order workflow execution.

## Pattern Recognition Notes

* **Asset Identity vs Instance Identity**: Verifying only the asset address is often insufficient when multiple lifecycle instances can exist over time. Operations should target a specific instance, not merely a resource identifier.

* **Stale State Execution**: External systems may act on observations made in the past. If state can change between observation and execution, freshness validation becomes critical.

* **Distributed System Assumptions**: Delayed, retried, or out-of-order messages should be considered normal behavior rather than exceptional conditions.

* **Replay Protection ≠ Freshness Protection**: Preventing duplicate successful executions does not prevent delayed first executions or stale retries.

* **Time-Based Economic State**: In Dutch auctions and similar mechanisms, elapsed time is part of the economic state. Resetting time can be economically harmful even when funds are not directly stolen.

* **Trust Boundary Validation**: When integrating with off-chain workflow systems, contracts must validate not only authenticity but also relevance and freshness.

## Quick Recall (TL;DR)

* **Bug**: CRE reports can be delivered late or retried, but `WorkflowRouter` performs no freshness validation.
* **Root Cause**: Auction end reports identify only the asset address, not the specific auction instance they were generated for.
* **Impact**: A stale "end auction" report can terminate a newly started auction for the same asset.
* **Why It Matters**: Inventory is returned, solver participation is interrupted, and Dutch auction price discovery resets.
* **Fix**: Bind reports to a unique auction instance (epoch/ID/start timestamp) and enforce report freshness before execution.
