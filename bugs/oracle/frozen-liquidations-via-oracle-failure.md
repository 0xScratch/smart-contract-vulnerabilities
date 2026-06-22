# Frozen Liquidations via Oracle Failure or Zero-Price Assets

* **Severity**: Medium
* **Source**: [Sherlock](https://github.com/sherlock-audit/2023-02-blueberry-judging/issues/161)
* **Affected Contracts**: [BlueBerryBank.sol](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol), [CoreOracle.sol](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/CoreOracle.sol), [ChainlinkAdapterOracle.sol](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/ChainlinkAdapterOracle.sol#L66-L84)
* **Vulnerability Type**: Denial of Service (DoS) / Oracle Dependency Risk / Liquidation Failure

## Summary

Blueberry's liquidation system depends on successfully obtaining oracle prices for every asset involved in a position. Before a liquidation can occur, the protocol calculates the position's risk by querying oracle prices for:

* The collateral asset
* The borrowed (debt) asset
* The isolated underlying collateral asset

If any of these price lookups fail, return stale data, or return a price of zero, the entire risk calculation reverts. Since liquidation eligibility depends on this risk calculation, all liquidation attempts revert as well.

As a result, during extreme market conditions—precisely when liquidations are most important—the protocol can become unable to liquidate unhealthy positions. This can allow underwater borrowers to avoid liquidation and potentially leave the protocol with bad debt.

## A Better Explanation (With Simplified Example)

### Intended Behavior

A lending protocol normally follows this process:

1. User deposits collateral.
2. User borrows assets against that collateral.
3. Market prices move.
4. Protocol continuously evaluates whether the position remains healthy.
5. If collateral value falls below safe limits, liquidators repay debt and seize collateral.

In Blueberry, liquidation follows:

```text
liquidate()
    ↓
isLiquidatable()
    ↓
getPositionRisk()
    ↓
Read Oracle Prices
    ↓
Calculate Risk
    ↓
Execute Liquidation
```

The protocol assumes oracle prices will always be available.

### What Actually Happens (Bug)

Blueberry cannot determine whether a position is liquidatable without first obtaining oracle prices.

If an oracle:

* Goes offline,
* Stops updating,
* Returns stale data,
* Returns a price of zero,

then the price lookup reverts.

That revert propagates through:

```text
getPrice()
    ↓
getDebtValue()
or
getCollateralValue()
    ↓
getPositionRisk()
    ↓
isLiquidatable()
    ↓
liquidate()
```

causing every liquidation attempt to fail.

As a result, the protocol effectively grants liquidation immunity to affected borrowers.

### Why This Matters

Oracle failures usually happen during highly volatile markets.

Examples:

* Stablecoin depegs
* Market crashes
* Oracle outages
* Exchange failures
* Liquidity crises

These are exactly the situations where liquidations must happen quickly to prevent insolvency.

Instead, Blueberry enters a state where:

```text
Collateral crashing
        +
Oracle unavailable
        =
Liquidations disabled
```

The protocol becomes unable to remove bad debt when it needs liquidations the most.

## Concrete Walkthrough (Alice & Bob)

### Initial State

Alice opens a leveraged position.

```text
Collateral Value = $1,000
Debt Value       = $700
```

Position is healthy.

### Market Crash

The collateral rapidly loses value.

```text
Collateral Value = $300
Debt Value       = $700
```

Alice should now be liquidatable.

Normally:

```text
Liquidator repays debt
    ↓
Receives collateral
    ↓
Protocol remains solvent
```

### Oracle Failure

During the crash, the collateral's Chainlink feed is paused.

When a liquidator attempts:

```solidity
liquidate(positionId, ...)
```

Blueberry executes:

```text
liquidate()
    ↓
isLiquidatable()
    ↓
getPositionRisk()
    ↓
oracle.getCollateralValue()
```

The oracle eventually reaches:

```solidity
registry.latestRoundData(...)
```

which fails or becomes stale.

The oracle reverts.

### Result

The entire transaction reverts.

```text
Liquidation Attempt
       ↓
Oracle Failure
       ↓
Risk Calculation Fails
       ↓
Liquidation Reverts
```

Alice remains in the system despite being underwater.

If collateral continues falling:

```text
Collateral = $50
Debt       = $700
```

the protocol accumulates unrecoverable bad debt.

## Vulnerable Code Reference

### 1) Liquidation depends on `isLiquidatable()`

```solidity
function liquidate(
    uint256 positionId,
    address debtToken,
    uint256 amountCall
) external override lock poke(debtToken) {
    if (!isLiquidatable(positionId))
        revert NOT_LIQUIDATABLE(positionId);
}
```

Every liquidation must successfully evaluate risk.

### 2) Risk calculation requires oracle prices

```solidity
function getPositionRisk(uint256 positionId)
    public
    view
    returns (uint256 risk)
{
    uint256 pv = getPositionValue(positionId);
    uint256 ov = getDebtValue(positionId);
    uint256 cv = oracle.getUnderlyingValue(
        pos.underlyingToken,
        pos.underlyingAmount
    );
}
```

All three values require oracle lookups.

### 3) Debt valuation depends on oracle pricing

```solidity
value += oracle.getDebtValue(token, debt);
```

A failed price lookup causes the entire call stack to revert.

### 4) CoreOracle rejects zero prices

```solidity
function _getPrice(address token)
    internal
    view
    returns (uint256)
{
    uint256 px =
        IBaseOracle(tokenSettings[token].route)
            .getPrice(token);

    if (px == 0)
        revert PRICE_FAILED(token);

    return px;
}
```

A token price of zero immediately reverts.

### 5) ChainlinkAdapter rejects stale prices

```solidity
if (updatedAt < block.timestamp - maxDelayTime)
    revert PRICE_OUTDATED(_token);
```

Any stale feed freezes downstream risk calculations.

### 6) Chainlink feed lookup can fail entirely

```solidity
registry.latestRoundData(
    token,
    USD
);
```

If the underlying oracle is paused or unavailable, the call may revert.

## Recommended Mitigation

### 1. Use Last Known Good Prices

Instead of reverting on stale oracle data:

```solidity
if (updatedAt < block.timestamp - maxDelayTime)
    revert PRICE_OUTDATED(_token);
```

store and use:

```solidity
lastGoodPrice[token]
```

for liquidation calculations.

This ensures liquidations remain possible during oracle disruptions.

### 2. Fail Conservatively

If collateral pricing fails:

```text
Collateral Value = 0
```

If debt pricing fails:

```text
Debt Value = Very Large
```

This biases the system toward liquidation rather than protecting risky borrowers.

### 3. Separate Borrowing from Liquidation Logic

Oracle failures should disable:

```text
Borrowing
Leverage Increases
New Positions
```

but should not disable:

```text
Liquidations
Debt Reduction
Risk Removal
```

### 4. Emergency Oracle Mechanism

Introduce governance or guardian controls that can:

* Switch oracle providers
* Activate backup feeds
* Override failed routes
* Restore liquidations during oracle outages

### 5. Add Oracle Failure Tests

Include tests for:

* Stale oracle data
* Zero-price responses
* Paused Chainlink feeds
* Reverting oracle calls

and verify that liquidations remain functional.

## Pattern Recognition Notes

### Critical Functions Must Not Depend on Fragile Inputs

Liquidations are safety mechanisms.

Any external dependency that can disable liquidations should be treated as high risk.

### Oracle Availability Risk

Many protocols focus on price manipulation.

Equally dangerous is oracle unavailability.

A perfectly accurate oracle is useless if its failure freezes critical protocol functions.

### Revert-Based Safety Can Become a Vulnerability

Developers often write:

```solidity
if (price == 0)
    revert;
```

to avoid bad pricing.

However, for liquidation systems this may create:

```text
Oracle Failure
       ↓
Protocol Failure
```

rather than preserving safety.

### Emergency Paths Matter

Every lending protocol should answer:

> "What happens if the oracle disappears tomorrow?"

If the answer is:

```text
Liquidations stop
```

then the protocol has a systemic solvency risk.

### Favor Solvency Over Precision

During crisis events, using a conservative estimate is often safer than refusing to operate.

The protocol's priority should be:

```text
Remove bad debt
      >
Perfect valuation
```

## Quick Recall (TL;DR)

* **Bug**: Liquidations require successful oracle pricing for all position assets.
* **Trigger**: Oracle outage, paused feed, stale data, or zero-price response.
* **Impact**: Risk calculation reverts → `isLiquidatable()` reverts → all liquidations fail.
* **Consequence**: Underwater positions become temporarily immune to liquidation, potentially creating protocol-wide bad debt and insolvency.
* **Fix**: Use fallback pricing (e.g., last-good-price), fail conservatively, and ensure oracle failures cannot disable liquidation mechanisms.
