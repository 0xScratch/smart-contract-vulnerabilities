# Shared Heartbeat Misconfiguration Across Chainlink Feeds Causes Oracle Downtime or Stale Price Acceptance

* **Severity**: Medium
* **Source**: [Sherlock](https://github.com/sherlock-audit/2023-04-jojo-judging/issues/449)
* **Affected Contract**: [chainlinkAdaptor.sol](https://github.com/sherlock-audit/2023-04-jojo/blob/main/smart-contract-EVM/contracts/adaptor/chainlinkAdaptor.sol#L43-L55)
* **Vulnerability Type**: Oracle Misconfiguration / Stale Price Validation / Availability Risk

## Summary

`chainlinkAdaptor` retrieves prices from two separate Chainlink feeds:

1. Asset/USD feed (e.g., ETH/USD)
2. USDC/USD feed

Before using the prices, the contract validates that both feeds are fresh by comparing their update timestamps against a single shared `heartbeatInterval`.

The issue is that different Chainlink feeds often have different heartbeat configurations. For example, an ETH/USD feed may have a heartbeat of 1 hour, while a USDC/USD feed may have a heartbeat of 24 hours.

Because the contract uses one heartbeat value for both feeds, the protocol is forced into an impossible tradeoff:

* Configure a short heartbeat and the USDC feed will frequently appear stale, causing oracle failures and protocol downtime.
* Configure a long heartbeat and the asset feed can become significantly stale while still passing validation.

As a result, the protocol can either become unavailable or consume stale oracle data.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The adapter wants to calculate an asset's price denominated in USDC.

Since Chainlink provides:

```
ETH/USD
USDC/USD
```

the contract computes:

```
ETH/USD
---------
USDC/USD
=
ETH/USDC
```

Before performing the calculation, both feeds should be checked to ensure their prices are fresh.

Ideally:

```
ETH/USD  -> validated using ETH heartbeat
USDC/USD -> validated using USDC heartbeat
```

### What Actually Happens (Bug)

Instead of using separate heartbeat values, the contract validates both feeds using the same variable:

```solidity
heartbeatInterval
```

```solidity
require(
    block.timestamp - updatedAt <= heartbeatInterval
);

require(
    block.timestamp - USDCUpdatedAt <= heartbeatInterval
);
```

This assumes that both Chainlink feeds have identical update schedules, which is often false.

### Why This Matters

Different Chainlink feeds are designed with different heartbeat intervals.

Example:

```
ETH/USD  -> 1 hour heartbeat
USDC/USD -> 24 hour heartbeat
```

A single heartbeat value cannot correctly validate both feeds at the same time.

The protocol must choose between:

* Rejecting valid data from one feed
* Accepting stale data from another feed

Both outcomes are dangerous.

## Concrete Walkthrough (Downtime Scenario)

Assume:

```
ETH/USD heartbeat  = 1 hour
USDC/USD heartbeat = 24 hours
```

Protocol configuration:

```solidity
heartbeatInterval = 1 hour;
```

### ETH Feed

Last update:

```
10:00
```

Current time:

```
10:30
```

Validation:

```solidity
30 minutes <= 1 hour
```

Passes.

### USDC Feed

Last update:

```
Yesterday 20:00
```

Current time:

```
Today 10:30
```

Validation:

```solidity
14.5 hours <= 1 hour
```

Fails.

The transaction reverts:

```solidity
"USDC_ORACLE_HEARTBEAT_FAILED"
```

Even though the USDC feed is behaving exactly as Chainlink intended.

Result:

```
Oracle unavailable
↓
Price retrieval fails
↓
Protocol functionality disrupted
```

## Concrete Walkthrough (Stale Price Scenario)

Developers realize the oracle keeps reverting and increase:

```solidity
heartbeatInterval = 24 hours;
```

Now the USDC feed works.

However:

### ETH Feed Stops Updating

Last update:

```
10:00
```

Current time:

```
22:00
```

Elapsed time:

```
12 hours
```

Validation:

```solidity
12 hours <= 24 hours
```

Passes.

The protocol now accepts an ETH price that is 12 hours old despite the feed's intended heartbeat being only 1 hour.

Result:

```
Stale ETH price accepted
↓
Incorrect mark prices
↓
Incorrect margin calculations
↓
Potential liquidation errors
```

## Why the Sponsor's Response Missed the Issue

The sponsor argued that heartbeat checks exist to ensure Chainlink continues updating.

While that statement is true, it does not address the reported issue.

The vulnerability is not that heartbeat validation exists.

The vulnerability is that:

```
Two different feeds
↓
Different heartbeat guarantees
↓
One shared heartbeat configuration
```

The stale-check logic itself is correct.

The configuration model is flawed because it assumes all feeds share identical heartbeat requirements.

## Analogy

Imagine a grocery store applying a single expiration rule to every product.

```
Milk      -> expires in 1 day
Canned food -> expires in 1 year
```

If the store chooses:

```
Expiration = 1 day
```

it throws away perfectly good canned food.

If the store chooses:

```
Expiration = 1 year
```

it considers spoiled milk safe to consume.

Neither choice is correct.

Each product requires its own expiration period.

Chainlink heartbeats work the same way.

Each feed requires its own freshness threshold.

## Vulnerable Code Reference

**Shared heartbeat used for two independent feeds**

```solidity
function getMarkPrice() external view returns (uint256 price) {
    int256 rawPrice;
    uint256 updatedAt;

    (, rawPrice, , updatedAt, ) =
        IChainlink(chainlink).latestRoundData();

    (, int256 USDCPrice,, uint256 USDCUpdatedAt,) =
        IChainlink(USDCSource).latestRoundData();

    require(
        block.timestamp - updatedAt <= heartbeatInterval,
        "ORACLE_HEARTBEAT_FAILED"
    );

    require(
        block.timestamp - USDCUpdatedAt <= heartbeatInterval,
        "USDC_ORACLE_HEARTBEAT_FAILED"
    );

    uint256 tokenPrice =
        (SafeCast.toUint256(rawPrice) * 1e8) /
        SafeCast.toUint256(USDCPrice);

    return tokenPrice * 1e18 / decimalsCorrection;
}
```

## Recommended Mitigation

### 1. Use Separate Heartbeats Per Feed (Primary Fix)

Store and validate each feed independently.

```solidity
uint256 assetHeartbeat;
uint256 usdcHeartbeat;
```

```solidity
require(
    block.timestamp - updatedAt <= assetHeartbeat,
    "ORACLE_HEARTBEAT_FAILED"
);

require(
    block.timestamp - USDCUpdatedAt <= usdcHeartbeat,
    "USDC_ORACLE_HEARTBEAT_FAILED"
);
```

### 2. Make Heartbeat Configuration Feed-Specific

Avoid globally shared oracle freshness parameters.

Instead, associate heartbeat values with the exact feed being queried.

```solidity
mapping(address => uint256) public feedHeartbeat;
```

This makes the system safer and easier to maintain when new feeds are added.

### 3. Add Oracle Configuration Tests

Add tests that simulate feeds with different heartbeat schedules.

Example:

* Feed A = 1 hour
* Feed B = 24 hours

Verify that both are accepted or rejected according to their individual configurations.

### 4. Document Oracle Assumptions

Operational documentation should explicitly state:

* Which feeds are used
* Expected heartbeat values
* Acceptable staleness thresholds

This reduces future misconfiguration risk.

## Pattern Recognition Notes

* **One-Size-Fits-All Oracle Validation**: Different oracles often have different update frequencies. Applying a shared freshness threshold is a common integration mistake.

* **Configuration-Level Vulnerabilities**: Sometimes the logic itself is correct, but a flawed configuration model makes secure operation impossible.

* **Oracle Availability vs Oracle Freshness Tradeoff**: If a single parameter governs multiple feeds with different characteristics, developers may be forced to choose between downtime and stale data acceptance.

* **Feed-Specific Assumptions Matter**: Auditors should never assume all Chainlink feeds share the same heartbeat. Verify each feed individually.

* **Staleness Checks Are Only As Good As Their Thresholds**: A correctly implemented freshness check becomes ineffective when paired with an incorrect staleness window.

## Quick Recall (TL;DR)

* **Bug**: Two Chainlink feeds with different heartbeat schedules are validated using a single shared `heartbeatInterval`.
* **Impact**:

  * Short heartbeat → valid feed data rejected → protocol downtime.
  * Long heartbeat → stale feed data accepted → incorrect pricing.
* **Root Cause**: Assumption that multiple Chainlink feeds share the same heartbeat requirements.
* **Fix**: Store and validate a separate heartbeat value for each oracle feed.
