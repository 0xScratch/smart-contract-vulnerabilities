# Auction Price Continues Decaying During Pause, Giving First Post-Unpause Bidder an Unearned Discount

* **Severity**: Medium
* **Source**: [Code4rena](https://code4rena.com/audits/2026-03-chainlink-payment-abstraction-v2/submissions/S-156)
* **Affected Contract**: [BaseAuction.sol](https://github.com/code-423n4/2026-03-chainlink/blob/main/src/BaseAuction.sol)
* **Vulnerability Type**: Business Logic Flaw / Auction Invariant Violation / Time-Based State Accounting

## Summary

The Dutch auction mechanism calculates price decay using:

```solidity
elapsedTime = block.timestamp - auctionStart;
```

However, `block.timestamp` continues advancing while the protocol is paused, even though bidding is completely disabled via `whenNotPaused`.

As a result, auction prices continue decaying during periods where no participant is allowed to bid. When the protocol is eventually unpaused, the first bidder receives a discount that accumulated during a period of zero price discovery.

The protocol therefore sells assets for less LINK than intended, violating the economic assumptions of the Dutch auction design.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The auction follows a standard Dutch auction model:

```text
High Price
    ↓
Nobody buys
    ↓
Price decreases
    ↓
Nobody buys
    ↓
Price decreases
    ↓
Eventually someone buys
```

The key assumption is:

> If the price decreases, it is because market participants had an opportunity to purchase the asset and chose not to.

In other words:

```text
Price Decay
=
Market Rejection
```

The discount exists because the market continuously rejects the current price.

### What Actually Happens (Bug)

The auction price is derived from elapsed time:

```solidity
uint256 elapsedTime = block.timestamp - auctionStart;
```

When the protocol is paused:

```solidity
function bid(...) external whenNotPaused
```

bidding becomes impossible.

However:

```text
Auction Clock     → Continues
Price Decay       → Continues
Bidding           → Disabled
```

This creates a mismatch:

```text
Time passes
↓
Price decreases
↓
Nobody could buy
```

The system behaves as though the market rejected the price, even though the market never had an opportunity to interact.

### Why This Matters

Dutch auctions rely on continuous price discovery.

During a pause:

```text
Price Discovery = Disabled
```

Therefore:

```text
Price Decay = Invalid
```

because the protocol is reducing prices under the assumption that participants are rejecting the auction price.

When the protocol is later unpaused, the first bidder receives a discount that was never earned through competition.

The protocol receives less LINK than it would have received had the auction clock stopped while bidding was unavailable.

### Concrete Walkthrough (Chainlink Auction Example)

Assume:

```text
Auction Asset: 100,000 USDC

Auction Duration: 24 hours

Starting Multiplier: 1.05x
Ending Multiplier:   0.99x
```

### Normal Scenario

Auction starts:

```text
Hour 0
Price = 1.05x
Cost = 5,250 LINK
```

One hour later:

```text
Hour 1
Price = 1.0475x
Cost = 5,237.5 LINK
```

Small discount.

Normal behavior.

### Pause Scenario

At Hour 1:

```text
Protocol Paused
```

Now:

```text
Bidding Disabled
```

Twenty hours pass.

Nobody can participate.

Nobody can buy.

Nobody can reject prices.

### Unpause

When the protocol resumes:

```solidity
elapsedTime =
    block.timestamp - auctionStart
```

The auction sees:

```text
Elapsed Time = 21 hours
```

even though only:

```text
1 hour
```

of actual bidding opportunity existed.

The auction price becomes:

```text
Price = 0.9975x
Cost = 4,987.5 LINK
```

instead of:

```text
Price = 1.0475x
Cost = 5,237.5 LINK
```

Difference:

```text
250 LINK
```

At:

```text
$20 / LINK
```

the protocol loses:

```text
$5,000
```

of value on a single auction.

### Why the First Bidder Benefits

Immediately after unpause:

```text
Auction already appears 21 hours old
```

The first bidder can purchase assets at the heavily discounted price.

They effectively capture:

```text
Discount accumulated during pause
```

despite no competitive bidding occurring during that period.

> **Analogy:** Imagine a store that lowers prices by $1 every hour until an item sells. The store closes for 20 hours. When it reopens, the price has dropped by $20 even though no customers were allowed inside. The first customer receives a discount that was never competed for.

## Vulnerable Code Reference

### 1) Raw Elapsed Time Includes Paused Duration

```solidity
uint256 auctionStart = s_auctionStarts[asset];

uint256 elapsedTime = block.timestamp - auctionStart;
```

Problem:

```text
block.timestamp continues advancing while paused
```

but:

```text
bidding is disabled
```

No paused duration is removed from the calculation.

### 2) Decay Curve Uses Inflated Elapsed Time

```solidity
elapsedTime = elapsedTime > assetInParams.auctionDuration
    ? assetInParams.auctionDuration
    : elapsedTime;
```

The value is merely capped.

It is not corrected.

### 3) Price Multiplier Decays Using Inflated Time

```solidity
uint256 priceMultiplier =
    assetInParams.startingPriceMultiplier
        - uint256(
            assetInParams.startingPriceMultiplier
            - assetInParams.endingPriceMultiplier
        ).mulDiv(
            elapsedTime,
            assetInParams.auctionDuration
        );
```

Larger elapsed time:

```text
↓ Lower Multiplier
↓ Cheaper Auction
↓ Less LINK Received
```

### 4) Bidding Is Explicitly Disabled During Pause

```solidity
function bid(...)
    external
    whenNotPaused
```

This confirms:

```text
Price Discovery = Impossible
```

during the period where price decay still occurs.

## Recommended Mitigation

### Option 1: Track Paused Duration (Recommended)

Store cumulative paused time and remove it from auction age calculations.

Example:

```solidity
uint256 elapsedTime =
    block.timestamp
    - auctionStart
    - pausedDuration;
```

Benefits:

* Preserves original auction start timestamp.
* Preserves off-chain indexing assumptions.
* Makes auction age reflect actual active bidding time.

### Option 2: Shift Auction Start Forward

Upon unpause:

```solidity
s_auctionStarts[asset] += pauseDuration;
```

This effectively pauses the auction clock.

Benefits:

* Simpler implementation.

Tradeoff:

* Auction start timestamps no longer match emitted events.
* Can confuse off-chain indexers.

### Additional Defensive Considerations

When designing pause mechanisms around time-sensitive logic:

* Explicitly define whether time should continue advancing while paused.
* Ensure auction timers, vesting timers, liquidation timers, and reward timers account for pause semantics.
* Test long-duration pauses against economic assumptions.

## Pattern Recognition Notes

### Auction Clock ≠ Wall Clock

A common mistake is assuming:

```text
Auction Age
=
block.timestamp - startTime
```

This is only correct when the auction is continuously accessible.

If participation is disabled:

```text
Auction Age
≠
Wall Clock Age
```

### Pause Mechanisms Often Break Time-Based Logic

Whenever you see:

```solidity
whenNotPaused
```

combined with:

```solidity
block.timestamp
```

ask:

> Should time continue progressing while interaction is impossible?

This pattern appears frequently in:

* Dutch auctions
* Liquidation systems
* Vesting schedules
* Reward accrual systems
* Streaming payments

### Economic Invariant Violations Can Be Medium Severity

No funds are directly stolen.

However:

```text
Protocol receives less value than intended.
```

This is often sufficient for Medium severity when the flaw breaks a core economic mechanism.

In this case:

```text
Continuous Dutch Auction Price Discovery
```

is the invariant that gets violated.

### Bounded Loss ≠ No Vulnerability

The auction has a minimum price floor:

```text
endingPriceMultiplier
```

which limits maximum loss.

However:

```text
Bounded Loss
≠
No Loss
```

A cap on damage reduces severity but does not eliminate the underlying flaw.

## Quick Recall (TL;DR)

* **Bug**: Auction price decay uses raw `block.timestamp`, so decay continues while the protocol is paused.
* **Root Cause**: Auction age includes periods where bidding was impossible.
* **Impact**: First post-unpause bidder receives an unearned discount; protocol receives less LINK than intended.
* **Invariant Broken**: Dutch auction price decay assumes active price discovery, which does not exist during pause.
* **Fix**: Exclude paused duration from elapsed-time calculations or shift auction start timestamps forward during unpause.
* **Key Lesson**: Whenever a protocol uses both `whenNotPaused` and time-based pricing, verify whether the clock should stop while interaction is disabled.
