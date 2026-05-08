# Unbounded `ownerOf()` Lookup Can Permanently DoS Raffle Winner Resolution

* **Severity**: Medium
* **Source**: [Sherlock](https://github.com/sherlock-audit/2024-08-winnables-raffles-judging/issues/398)
* **Affected Contracts**:

  * [WinnablesTicket.sol](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/contracts/WinnablesTicket.sol#L97-L99)
  * [WinnablesTicketManager.sol](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/contracts/WinnablesTicketManager.sol)
* **Vulnerability Type**: Denial of Service (DoS) / Unbounded Iteration / Gas Exhaustion

## Summary

`WinnablesTicket` uses an ERC721A-style compressed ownership model where only the first ticket in a minted batch stores ownership information. All subsequent ticket IDs in the batch remain unset (`address(0)`).

To resolve ownership of a ticket, `ownerOf()` linearly scans backward until it finds the nearest initialized ownership slot.

This optimization makes minting cheap, but causes ownership resolution to become increasingly expensive as ticket batch sizes grow.

If a user purchases a sufficiently large ticket batch (for example ~4000+ tickets), and the randomly selected winning ticket lies near the end of that batch, the backward scan inside `ownerOf()` may consume excessive gas and revert with Out-of-Gas (OOG).

Because raffle winner propagation depends on successfully resolving the winner ticket ownership, the raffle may become permanently stuck, locking prize distribution and protocol funds.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The protocol wants ticket minting to be cheap.

Instead of storing ownership for every ticket individually:

```text
Ticket 0 -> Alice
Ticket 1 -> Alice
Ticket 2 -> Alice
Ticket 3 -> Alice
...
```

the contract stores ownership only once at the beginning of the batch:

```text
Ticket 0 -> Alice
Ticket 1 -> empty
Ticket 2 -> empty
Ticket 3 -> empty
...
```

This greatly reduces storage writes during minting.

To determine ownership later, `ownerOf()` walks backward until it finds the nearest initialized owner slot.

## What Actually Happens (Bug)

Suppose Alice buys 4000 tickets.

The contract stores only:

```text
_ticketOwnership[0] = Alice
```

Now imagine the winning ticket is:

```text
3999
```

When the protocol calls:

```solidity
ownerOf(raffleId, 3999)
```

the function performs:

```text
3999 -> empty
3998 -> empty
3997 -> empty
...
0 -> Alice
```

This requires thousands of storage reads and loop iterations.

Eventually:

* gas usage becomes extremely high,
* block gas limits may be exceeded,
* the transaction reverts with OOG.

The protocol cannot resolve the winner anymore.

## Why This Matters

The issue affects a critical raffle settlement path.

Winner propagation flow:

1. Users buy tickets.
2. Chainlink VRF returns randomness.
3. Protocol computes winning ticket.
4. Protocol resolves ticket owner via `ownerOf()`.
5. Prize is propagated cross-chain.

If step 4 fails due to OOG:

* raffle cannot finalize,
* winner cannot be propagated,
* ETH remains locked,
* prize distribution stalls.

The protocol effectively enters a DoS state for that raffle.

## Concrete Walkthrough (Alice Scenario)

### Setup

Alice purchases 4000 tickets.

Internally:

```text
_ticketOwnership[0] = Alice
_supply = 4000
```

No ownership is stored for tickets `1 → 3999`.

### Winner Selection

Chainlink VRF returns randomness:

```solidity
winningTicket = randomWord % supply;
```

Suppose:

```text
winningTicket = 3999
```

### Dangerous Lookup

Protocol executes:

```solidity
ownerOf(raffleId, 3999)
```

Inside `ownerOf()`:

```solidity
while (_ticketOwnership[id][ticketId] == address(0)) {
    --ticketId;
}
```

The function scans backward:

```text
3999 -> empty
3998 -> empty
3997 -> empty
...
0 -> Alice
```

Thousands of storage reads occur.

Gas consumption becomes extremely large.

Eventually:

* transaction exceeds practical gas limits,
* call reverts with OOG.

### Result

The raffle cannot successfully propagate the winner.

Effects:

* winner resolution fails,
* raffle settlement stalls,
* locked ETH cannot progress,
* prize distribution freezes.

> **Analogy**:
>
> Imagine a stadium where only the first seat in each section has the owner's name written down.
>
> If someone asks:
>
> "Who owns seat 3999?"
>
> the staff walks backward seat-by-seat checking every chair until eventually reaching seat 0.
>
> For sufficiently large sections, the lookup itself becomes too expensive and the process collapses.

## Vulnerable Code Reference

### 1) Compressed Ownership Storage During Mint

```solidity
function mint(address to, uint256 id, uint256 amount) external onlyRole(1) {
    uint256 startId = _supplies[id];

    unchecked {
      _balances[id][to] += amount;
      _supplies[id] = startId + amount;
    }

    _ticketOwnership[id][startId] = to;
}
```

Only the first ticket in the batch stores ownership.

### 2) Unbounded Backward Scan in `ownerOf()`

```solidity
function ownerOf(uint256 id, uint256 ticketId) public view returns (address) {
    if (ticketId >= _supplies[id]) {
      revert InexistentTicket();
    }

    while (_ticketOwnership[id][ticketId] == address(0)) {
      unchecked { --ticketId; }
    }

    return _ticketOwnership[id][ticketId];
}
```

Gas complexity grows linearly with ticket distance from initialized ownership slot.

### 3) Winner Resolution Depends on `ownerOf()`

```solidity
function _getWinnerByRequestId(uint256 requestId) internal view returns(address) {
    RequestStatus storage request = _chainlinkRequests[requestId];

    uint256 supply =
        IWinnablesTicket(TICKETS_CONTRACT).supplyOf(request.raffleId);

    uint256 winningTicketNumber =
        request.randomWord % supply;

    return IWinnablesTicket(TICKETS_CONTRACT)
        .ownerOf(request.raffleId, winningTicketNumber);
}
```

A failing `ownerOf()` blocks raffle settlement.

## Recommended Mitigation

### 1) Enforce Reasonable Ticket Batch Limits

Prevent excessively large ownership gaps:

```solidity
require(ticketCount <= MAX_SAFE_BATCH_SIZE);
```

This was the sponsor's intended operational mitigation.

### 2) Store Ownership More Frequently

Instead of storing ownership only once per batch:

```text
0 -> Alice
100 -> Alice
200 -> Alice
...
```

This bounds maximum scan distance.

### 3) Use O(1) Ownership Tracking

Avoid linear backward traversal entirely.

Possible approaches:

* explicit ownership mapping per ticket,
* bitmap indexing,
* bounded checkpointing,
* alternative compressed ownership structures.

### 4) Add Gas-Bounded Resolution Guarantees

Critical settlement paths should never rely on:

* unbounded loops,
* user-controlled iteration size,
* storage-dependent scans.

Especially in:

* winner resolution,
* withdrawals,
* liquidation,
* settlement logic.

## Pattern Recognition Notes

* **Write Optimization vs Read Explosion**:
  Saving gas during writes can accidentally create extremely expensive reads later.

* **Unbounded Iteration on User-Controlled State**:
  Any loop whose size depends on user-controlled input is dangerous in critical execution paths.

* **ERC721A-Style Ownership Compression Risks**:
  Compressed ownership patterns require careful analysis of worst-case lookup costs.

* **Gas Complexity Matters**:
  Even view-like helper functions become dangerous when used inside state-changing protocol logic.

* **Critical Paths Must Be Gas Predictable**:
  Raffle settlement, liquidation, and withdrawal logic should never rely on potentially unbounded scans.

* **Off-Chain Assumptions Are Fragile**:
  If protocol safety depends on backend/API restrictions, those assumptions should also be enforced on-chain.

## Quick Recall (TL;DR)

* **Bug**: `ownerOf()` performs unbounded backward scanning through sparse ownership slots.
* **Trigger**: Large ticket batches + winning ticket near batch end.
* **Impact**: OOG during winner resolution → raffle settlement DoS.
* **Root Cause**: ERC721A-style ownership compression creates O(n) ownership lookup complexity.
* **Fix**: Bound ticket batch size or redesign ownership tracking for O(1) lookups.
