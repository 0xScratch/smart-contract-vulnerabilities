# Bad Debt Accumulation via WBTC Depeg Oracle Assumption

* **Severity**: Medium
* **Source**: [Sherlock](https://github.com/sherlock-audit/2023-05-USSD-judging/issues/310)
* **Affected Contract**: [StableOracleWBTC.sol](https://github.com/sherlock-audit/2023-05-USSD/blob/6d7a9fdfb1f1ed838632c25b6e1b01748d0bafda/ussd-contracts/contracts/oracles/StableOracleWBTC.sol#L12-L26)
* **Vulnerability Type**: Oracle Risk / Asset Peg Assumption / Collateral Mispricing

## Summary

The protocol prices WBTC using a **BTC/USD Chainlink oracle**, implicitly assuming that **1 WBTC will always equal 1 BTC**.

This assumption is normally valid because WBTC is designed to be backed 1:1 by Bitcoin. However, if WBTC ever depegs from BTC due to a custodian failure, bridge compromise, reserve issue, or loss of market confidence, the protocol would continue valuing WBTC at the BTC price even though the market value of WBTC has fallen significantly.

As a result, users could deposit devalued WBTC as collateral and borrow far more than the collateral is actually worth, creating bad debt for the protocol.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The protocol wants to determine:

```text
What is 1 WBTC worth in USD?
```

Since WBTC is expected to track Bitcoin:

```text
1 WBTC ≈ 1 BTC
```

the protocol simply reads the BTC/USD Chainlink oracle and treats that value as the price of WBTC.

For example:

```text
BTC = $30,000
WBTC = $30,000
```

Everything works correctly.

### What Actually Happens (Bug)

The protocol assumes:

```text
WBTC price = BTC price
```

instead of verifying:

```text
WBTC market price
```

If WBTC ever depegs, this assumption breaks.

Imagine a catastrophic event:

* WBTC custodian gets hacked
* Backing reserves disappear
* Bridge is compromised
* Market loses confidence

Now the market values WBTC at:

```text
BTC = $30,000
WBTC = $10,000
```

However, the protocol still reads:

```text
BTC/USD = $30,000
```

and therefore believes:

```text
WBTC = $30,000
```

The protocol is now overvaluing collateral by 3x.

### Why This Matters

Lending systems rely on accurate collateral valuation.

If collateral is worth less than the protocol believes:

* Users can borrow more than they should.
* Liquidations become ineffective.
* The protocol accumulates bad debt.
* Insolvency risk increases.

The most dangerous part is that the protocol does not recognize the depeg and therefore continues issuing undercollateralized loans.

### Concrete Walkthrough (Alice & Mallory)

#### Normal Scenario

Market:

```text
BTC = $30,000
WBTC = $30,000
```

Alice deposits:

```text
1 WBTC
```

Protocol values collateral at:

```text
$30,000
```

Assuming a 70% LTV:

```text
Maximum borrow = $21,000
```

This is safe.

---

#### Depeg Scenario

A WBTC reserve failure occurs.

Market now says:

```text
BTC = $30,000
WBTC = $10,000
```

Alice buys 1 WBTC for:

```text
$10,000
```

and deposits it into the protocol.

However, the protocol still uses BTC/USD and therefore values the collateral as:

```text
$30,000
```

Borrow limit becomes:

```text
70% × $30,000 = $21,000
```

Alice borrows:

```text
$21,000
```

worth of protocol assets.

She can immediately sell those borrowed assets and walk away.

The protocol now holds:

```text
Collateral = $10,000
Debt = $21,000
```

Result:

```text
Bad Debt = $11,000
```

for a single position.

### Why Bad Debt Keeps Growing

The protocol never notices that WBTC has depegged.

It continues using:

```text
BTC/USD
```

instead of:

```text
WBTC/USD
```

or another mechanism that verifies WBTC's actual market value.

Therefore:

```text
User 1 deposits WBTC
→ Borrows too much

User 2 deposits WBTC
→ Borrows too much

User 3 deposits WBTC
→ Borrows too much
```

Bad debt continues accumulating until the protocol becomes insolvent or borrowing is manually disabled.

> **Analogy**: Imagine a bank accepting counterfeit gold certificates while still valuing them as real gold. As long as the bank refuses to verify whether the certificates are actually redeemable, people can continuously borrow real money against nearly worthless paper.

## Vulnerable Code Reference

### WBTC Valued Using BTC/USD Oracle

```solidity
contract StableOracleWBTC is IStableOracle {
    AggregatorV3Interface priceFeed;

    constructor() {
        priceFeed = AggregatorV3Interface(
            0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419
        );
    }

    function getPriceUSD() external view override returns (uint256) {
        (, int256 price, , , ) = priceFeed.latestRoundData();
        return uint256(price) * 1e10;
    }
}
```

The oracle retrieves:

```text
BTC/USD
```

and assumes:

```text
WBTC/USD = BTC/USD
```

without verifying whether WBTC remains properly pegged.

## Root Cause

The protocol conflates:

```text
Underlying Asset Value (BTC)
```

with

```text
Wrapped Asset Value (WBTC)
```

These values are only equivalent while the peg remains intact.

The protocol assumes:

```text
WBTC == BTC
```

instead of validating:

```text
WBTC ≈ BTC
```

which creates systemic risk during depeg events.

## Recommended Mitigation

### 1. Use a Secondary Market-Based Oracle

Supplement the BTC/USD Chainlink feed with an oracle that reflects the actual market value of WBTC.

Examples:

* Uniswap V3 TWAP
* Curve TWAP
* Other liquidity-based price feeds

This provides visibility into WBTC's real trading value.

### 2. Implement Oracle Deviation Checks

Compare:

```text
BTC/USD
```

against:

```text
WBTC market price
```

If deviation exceeds a threshold:

```text
2%
5%
10%
```

the protocol should automatically pause borrowing.

Example:

```solidity
if (deviation > threshold) {
    pauseBorrowing();
}
```

### 3. Add Circuit Breakers

Disable:

* Borrowing
* New positions
* Collateral deposits

when severe oracle divergence is detected.

### 4. Risk-Based Collateral Management

Assets that depend on custodians, bridges, or wrappers should receive additional risk treatment:

* Lower LTVs
* Borrow caps
* Emergency pause functionality

## Pattern Recognition Notes

* **Wrapped Asset Assumption Risk**: Never assume a wrapped asset will permanently track its underlying asset.
* **Oracle Dependency Risk**: Correct oracle data can still produce incorrect protocol behavior if the wrong asset is being priced.
* **Collateral Mispricing**: Overvalued collateral allows users to extract more value than they contribute.
* **External Trust Assumptions**: Custodial and bridge-backed assets introduce risks that pure on-chain assets do not.
* **Systemic Insolvency Pattern**: When collateral valuation is inflated, every new loan increases protocol-wide bad debt.

## Quick Recall (TL;DR)

* **Bug**: WBTC is priced using BTC/USD, assuming WBTC always equals BTC.
* **Issue**: If WBTC depegs, the protocol continues overvaluing WBTC collateral.
* **Impact**: Users can borrow more than their collateral is actually worth, creating bad debt.
* **Why Medium?** Exploitation requires an external event (WBTC depeg) before the protocol can be abused.
* **Fix**: Use multiple oracle sources, monitor peg deviations, and halt borrowing during significant depegs.
