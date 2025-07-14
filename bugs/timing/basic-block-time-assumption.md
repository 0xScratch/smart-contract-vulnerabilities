# Immutable BLOCK_TIME Parameter Cause Oracle Flaws

- **Severity**: Medium
- **Source**: [Code4rena](https://github.com/code-423n4/2023-07-basin-findings/issues/176) / [One Bug Per Day](https://www.onebugperday.com/v1/324)
- **Affected Contract**: [MultiFlowPump.sol](https://github.com/code-423n4/2023-07-basin/blob/c1b72d4e372a6246e0efbd57b47fb4cbb5d77062/src/pumps/MultiFlowPump.sol#L39)
- **Vulnerability Type**: Timing / Time Manipulation

## Original Bug Description

> **Treating of BLOCK_TIME as permanent will cause serious economic flaws in the oracle when block times change**
>
> **Vulnerability Details**
>
> **Description**
>
> Treating of BLOCK_TIME as permanent will cause serious economic flaws in the oracle when block times change `blocksPassed` variable, which determines what is the maximum change in price (done in `_capReserve()`).
>
> The issue is that BLOCK_TIME is an immutable variable in the pump, which is immutable in the Well, meaning it is basically set in stone and can only be changed through a Well redeploy and liquidity migration (very long cycle). However, BLOCK_TIME actually changes every now and then, especially in L2s.For example, the recent Bedrock upgrade in Optimism completely [changed](https://community.optimism.io/docs/developers/bedrock/differences/#the-evm) the block time generation. It is very clear this will happen many times over the course of Basin's lifetime.
>
>When a wrong BLOCK_TIME is used, the `_capReserve()` function will either limit price changes too strictly, or too permissively. In the too strict case, this would cause larger and large deviations between the oracle pricing and the real market prices, leading to large arb opportunities. In the too permissive case, the function will not cap changes like it is meant too, making the oracle more manipulatable than the economic model used when deploying the pump.
>
> **Impact**
>
> Treating of BLOCK_TIME as permanent will cause serious economic flaws in the oracle when block times change.
>
> **Tools Used**: Manual Audit
>
> **Recommended Mitigation Steps**
>
> The BLOCK_TIME should be changeable, given a long enough freeze period where LPs can withdraw their tokens if they are unsatisfied with the change.

## Summary

A critical timing assumption was made by treating `BLOCK_TIME` as a permanent, immutable value. When actual block times change due to network upgrades (such as Optimism's Bedrock upgrade), this causes the `_capReserve()` function to calculate incorrect `blocksPassed` values, leading to either overly restrictive price caps (creating arbitrage opportunities) or overly permissive caps (enabling oracle manipulation).

## Real-World Context

This vulnerability is particularly relevant for:

- **Layer 2 networks** (Optimism, Arbitrum, Polygon) that frequently undergo upgrades
- **Oracle-dependent protocols** where price accuracy is critical
- **Cross-chain deployments** where block times vary significantly between networks

## Key Details

- **Pattern**: Hardcoded blockchain parameters treated as permanent.
- **Root Cause**: `BLOCK_TIME` set as immutable in the contract constructor.
- **Risk**: Oracle price manipulation, arbitrage, and economic instability if block time changes after deployment

## Vulnerable Code Reference

```solidity
uint256 immutable BLOCK_TIME;  // Set in constructor, never changeable (Line 39)
...
blocksPassed = (deltaTimestamp / BLOCK_TIME).fromUInt(); // (Line 113)
```

## Impact

Same as provided in the [Original Bug Description](#original-bug-description).

## Mitigation

Same as provided in the [Original Bug Description](#original-bug-description).

## Pattern Recognition Notes

- Watch for any immutable or constant blockchain parameters (block time, gas price, etc.) used in economic calculations
- Always consider if these parameters could change due to future protocol or network upgrades
- **Red flags**: `immutable` keyword used with blockchain-specific constants
- **Common locations**: Constructor parameters, time-based calculations, rate limiting logic
- **Testing approach**: Simulate different block times to verify system behavior
