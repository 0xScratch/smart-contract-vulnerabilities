# Manipulatable Spot Oracle Causes False Liquidations During Oracle Divergence

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/price_feed/CoreSaltyFeed.sol#L32-L53)
* **Affected Contract**: [CoreSaltyFeed.sol](https://github.com/code-423n4/2024-01-salty/blob/53516c2cdfdfacb662cdea6417c52f23c94d5b5b/src/price_feed/CoreSaltyFeed.sol#L32-L53)
* **Vulnerability Type**: Oracle Manipulation / Price Feed Composition Failure / Economic Attack

## Summary

`CoreSaltyFeed` derives BTC and ETH prices directly from the instantaneous reserve ratio of Salty liquidity pools.

```solidity
price = reservesUSDS / reservesAsset;
```

Because this is a pure AMM spot price, it can be manipulated through swaps. While Salty relies on arbitrage to eventually restore correct pricing, the oracle remains temporarily manipulatable.

The vulnerability becomes exploitable when multiple oracle sources disagree due to different update speeds. During a market movement, Chainlink may update immediately while TWAP remains stale. An attacker can then cheaply manipulate the `CoreSaltyFeed` spot price further in the desired direction, causing the aggregated protocol price to become artificially low.

This allows positions that are actually healthy to appear undercollateralized and become liquidatable, enabling the attacker to collect liquidation rewards.

## A Better Explanation (With Simplified Example)

### Intended Behavior

Salty uses multiple price sources:

1. Chainlink
2. TWAP
3. CoreSaltyFeed

The idea is that combining several oracles should provide a more robust price than relying on a single source.

For example:

```text
Real BTC Price = $40,000

Chainlink     = $40,000
TWAP          = $40,000
CoreSaltyFeed = $40,000
```

All sources agree, so the aggregated price is correct.

### What Actually Happens (Bug)

The problem is that the three oracle sources update at different speeds.

Suppose BTC suddenly drops by 3%.

```text
$40,000 → $38,800
```

#### Chainlink

Updates quickly:

```text
Chainlink = $38,800
```

#### TWAP

Still contains historical prices:

```text
TWAP = $40,000
```

#### CoreSaltyFeed

Arbitrage naturally moves Salty pools toward the new market price:

```text
CoreSaltyFeed = $38,800
```

At this point, there is already disagreement:

```text
Chainlink = $38,800
TWAP      = $40,000
```

The attacker now performs a small swap to push the AMM spot price slightly lower:

```text
CoreSaltyFeed = $38,000
```

Result:

```text
Chainlink     = $38,800
TWAP          = $40,000
CoreSaltyFeed = $38,000
```

The protocol now observes an artificially pessimistic oracle set.

### Why This Matters

The attacker does not need to create a huge fake price.

The market has already moved.

The TWAP is already stale.

The attacker only needs to slightly push the manipulatable oracle further in the desired direction.

This is much cheaper than performing a full-scale oracle manipulation.

The report demonstrates that moving the spot price by approximately 3% can cost only:

```text
~0.0036 ETH
```

for pools containing roughly 1000 ETH of liquidity.

## Concrete Walkthrough (Alice & Mallory)

### Setup

Alice has a borrowing position.

```text
Collateral Value = $10,100
Debt             = $10,000
Collateral Ratio = 101%
```

Assume liquidation occurs below 100%.

Alice is safe.

### Market Movement

BTC falls:

```text
$40,000 → $38,800
```

Chainlink updates immediately:

```text
$38,800
```

TWAP remains stale:

```text
$40,000
```

### Mallory Attack

Mallory manipulates the Salty pool.

Instead of:

```text
CoreSaltyFeed = $38,800
```

she pushes it to:

```text
CoreSaltyFeed = $38,000
```

The manipulation cost is extremely small relative to liquidation rewards.

### Oracle Aggregation

The protocol now computes a lower price than reality.

Example:

```text
(38,000 + 38,800) / 2
= $38,400
```

instead of:

```text
$38,800
```

### Result

Alice's collateral is now valued using the manipulated oracle.

```text
Collateral Ratio

Real Price:
101%

Manipulated Price:
99%
```

Alice suddenly appears liquidatable.

Mallory immediately calls liquidation and receives the liquidation reward.

Alice loses part of her collateral despite being healthy under the real market price.

> **Analogy:** Imagine three judges estimating the weight of a box. One judge updates instantly, one uses yesterday's measurements, and one can be bribed slightly. If the final decision uses all three opinions, the attacker only needs to influence the bribable judge a little because the outdated judge is already pulling the average in the wrong direction.

## Vulnerable Code Reference

### 1) CoreSaltyFeed Uses Direct Pool Reserves

BTC price:

```solidity
function getPriceBTC() external view returns (uint256)
{
    (uint256 reservesWBTC, uint256 reservesUSDS) =
        pools.getPoolReserves(wbtc, usds);

    return (reservesUSDS * 10**8) / reservesWBTC;
}
```

ETH price:

```solidity
function getPriceETH() external view returns (uint256)
{
    (uint256 reservesWETH, uint256 reservesUSDS) =
        pools.getPoolReserves(weth, usds);

    return (reservesUSDS * 10**18) / reservesWETH;
}
```

The oracle price is entirely determined by current reserves.

Any successful swap changes those reserves.

Therefore:

```text
Swap → Reserve Change → Oracle Price Change
```

### 2) Oracle Is Based on Spot Price

The oracle performs no:

```text
- TWAP calculation
- Time delay
- Manipulation resistance
- Price smoothing
```

It simply returns the current AMM state.

### 3) Aggregation Assumes CoreSaltyFeed Is Trustworthy

The attack becomes possible because the protocol combines:

```text
Chainlink
TWAP
CoreSaltyFeed
```

without accounting for the fact that CoreSaltyFeed can be actively manipulated while the other sources are temporarily stale.

## Recommended Mitigation

### 1. Do Not Use Manipulatable Spot Prices Directly

Avoid using instantaneous AMM reserve ratios as oracle inputs for liquidation-critical logic.

Instead:

```text
Use TWAP
Use external oracle networks
Use manipulation-resistant feeds
```

### 2. Add Oracle Deviation Checks

Before accepting a spot price:

```text
If |CoreSaltyFeed - Chainlink| > threshold

Reject the value
or
Reduce its weight
```

This prevents manipulated spot prices from influencing liquidations.

### 3. Use Median-Based Aggregation

Prefer:

```text
Median(Chainlink, TWAP, CoreSaltyFeed)
```

instead of averaging.

A median is significantly harder to manipulate because one malicious oracle cannot pull the result arbitrarily.

### 4. Introduce Circuit Breakers

Pause liquidations whenever oracle disagreement exceeds a predefined threshold.

Example:

```text
If oracle deviation > 2%

Disable liquidations temporarily
```

This prevents liquidations during periods of oracle instability.

### 5. Require Time-Weighted Pool Pricing

If AMM pricing must be used:

```text
Use TWAP over several minutes
```

rather than instantaneous reserve ratios.

This significantly increases manipulation costs.

## Pattern Recognition Notes

* **Spot Price Oracle Risk**:** AMM spot prices are not oracle-safe by default. Any swap changes the reported value.

* **Oracle Speed Mismatch:** Multiple oracle sources often update at different rates. During market volatility, this creates temporary disagreement that attackers can exploit.

* **Manipulate the Weakest Oracle:** Attackers usually do not need to corrupt every oracle source. They only need to influence the cheapest one enough to affect the aggregate result.

* **Aggregation Does Not Equal Security:** Combining several oracle feeds does not automatically make the system secure. The security of the aggregate is limited by its weakest component.

* **Liquidation-Sensitive Systems Require Strong Oracle Guarantees:** Any oracle feeding collateral calculations must be highly resistant to short-term manipulation because even small pricing errors can trigger liquidations.

* **Market Moves Create Attack Windows:** Many oracle attacks become dramatically cheaper immediately after large market movements because natural oracle disagreement already exists.

## Quick Recall (TL;DR)

* **Bug:** `CoreSaltyFeed` uses a directly manipulatable AMM spot price.
* **Attack:** During normal market movements, Chainlink updates immediately while TWAP lags. The attacker slightly manipulates the spot oracle further in the desired direction.
* **Impact:** Aggregated oracle price becomes artificially low, causing healthy positions to appear undercollateralized and become liquidatable.
* **Profit Source:** Liquidation rewards exceed the manipulation cost.
* **Fix:** Avoid spot-price oracles for liquidation logic; use TWAPs, deviation checks, median aggregation, and manipulation-resistant feeds.
