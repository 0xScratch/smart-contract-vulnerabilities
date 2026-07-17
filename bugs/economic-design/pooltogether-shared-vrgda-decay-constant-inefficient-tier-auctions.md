# Shared VRGDA Decay Constant Causes Smaller Prize Tier Auctions to Reach Maximum Fees Prematurely

* **Severity:** Medium
* **Source:** [Code4rena](https://github.com/code-423n4/2023-08-pooltogether-mitigation-findings/issues/15) / [One Bug Per Day](https://www.onebugperday.com/v1/504)
* **Affected Contract:** [Claimer.sol](https://github.com/GenerationSoftware/pt-v5-claimer/blob/main/src/Claimer.sol)
* **Vulnerability Type:** Economic Design / Auction Parameter Mismatch

## Summary

PoolTogether uses a **VRGDA (Variable Rate Gradual Dutch Auction)** to determine the fee paid to bots that claim prizes on behalf of winners.

Originally, all prize tiers shared the same maximum fee. This caused an issue where large prizes could become economically unattractive to claim if gas costs exceeded the fee.

To solve that problem, the protocol changed the design so that **each prize tier has its own maximum fee**, proportional to that tier's prize size.

However, the VRGDA auction itself was **not fully separated per tier**. While the maximum fee became tier-specific, all tiers continued sharing the same auction behavior (decay constant).

As a result, auctions for smaller prize tiers reached their maximum fee much sooner than intended, causing the protocol to overpay claimers even when they would have accepted lower fees.

## Simplified Explanation

Imagine the protocol hires bots to collect prizes for winners.

Instead of paying bots a fixed amount, the protocol starts with a small reward and gradually increases it until a bot decides the reward is worth the gas cost.

For example:

```
Minute 0   -> $1
Minute 10  -> $2
Minute 20  -> $3
Minute 30  -> $4
```

If a bot is willing to work for **$2**, the protocol only pays **$2**, instead of always paying the maximum.

This minimizes protocol costs while still ensuring prizes eventually get claimed.

---

Now suppose there are different prize sizes.

| Prize | Maximum Claim Fee |
| ----: | ----------------: |
| $1000 |              $200 |
|  $200 |               $40 |
|   $20 |                $4 |

This is reasonable because larger prizes can support larger claim incentives.

The problem is that although every tier now has a different maximum fee, they all still share the **same VRGDA auction behavior**.

This causes smaller fee ranges to reach their maximum much earlier than larger ones.

Consequently, bots often receive the maximum reward for small-prize claims even though they would have accepted much less.

## Intended Behavior

Each prize tier should have an auction that:

* Starts from a low claim fee.
* Gradually increases over the configured auction duration.
* Reaches its own maximum fee near the configured target time.
* Pays only the minimum amount necessary to incentivize a claimer.

In other words, each tier should progress through **its own fee range** at roughly the intended pace.

## Actual Behavior

Only the maximum fee became tier-specific.

The auction's decay constant remained shared across all prize tiers.

Therefore:

* Large prize tiers progressed normally.
* Small prize tiers exhausted their fee range much more quickly.
* Their auction effectively hit the maximum fee early.
* Claimers received larger rewards than economically necessary.

## Alice & Bob Example

Suppose the protocol has two prize tiers.

### Tier 1

```
Prize = $1000
Fee Range = $20 → $100
```

### Tier 2

```
Prize = $20
Fee Range = $1 → $4
```

Assume both auctions share the exact same VRGDA decay constant.

After some time:

* Tier 1's fee has naturally increased to around **$60**.
* Tier 2 has already reached its maximum fee of **$4**.

However, a claiming bot may have happily accepted **$2.50**.

Instead, because Tier 2's auction reached its cap prematurely, the protocol pays the full **$4**.

The claim succeeds, but the protocol spends more than necessary.

## Root Cause

The protocol changed the **maximum claim fee** to depend on prize size but continued using a **single shared VRGDA decay constant** for every prize tier.

Different fee ranges therefore shared the same auction progression, even though each range required different auction dynamics.

In short:

* **Fee caps became tier-specific.**
* **Auction behavior did not.**

This mismatch caused auctions for smaller prize tiers to complete much sooner than intended.

## Vulnerable Code Reference

The maximum fee is computed per prize tier:

```solidity
function _computeMaxFee(uint8 _tier) internal view returns (uint256) {
    uint256 prizeSize = prizePool.getTierPrizeSize(_tier);
    return convert(
        maxFeePortionOfPrize.intoUD60x18().mul(convert(prizeSize))
    );
}
```

However, the auction uses a shared decay constant when computing claim fees:

```solidity
decayConstant = LinearVRGDALib.getDecayConstant(...);
```

The same decay constant is reused for every tier, despite each tier having a different maximum fee.

## Impact

This issue does **not** allow an attacker to steal funds or manipulate prize claims.

Instead, it makes the fee market economically inefficient.

As a result:

* Small-prize auctions hit their fee cap prematurely.
* Claimers receive higher rewards than necessary.
* The protocol pays more incentives than required.
* Protocol funds are used less efficiently over time.

## Recommended Mitigation

Each prize tier should have its own auction parameters rather than sharing a single auction configuration.

In particular:

* Compute a separate decay constant for each tier.
* Allow each auction to progress through its own fee range independently.
* Ensure every tier reaches its own maximum fee over approximately the intended auction duration.

This keeps fee growth proportional across all prize tiers and prevents premature fee saturation.

## Pattern Recognition Notes

When reviewing auction-based or incentive-based protocols, look for situations where:

* Different markets share one pricing curve despite having different value ranges.
* Maximum values become asset-specific while pricing dynamics remain global.
* A fix adjusts price limits but leaves the auction mechanics unchanged.
* Multiple asset classes reuse the same economic parameters even though their scales differ.

These often lead to **economic inefficiencies** rather than direct exploits.

## Quick Recall (TL;DR)

* **Bug:** Tier-specific maximum fees were introduced, but all tiers still shared one VRGDA decay constant.
* **Root Cause:** Fee caps became independent while auction progression remained shared.
* **Impact:** Smaller prize tiers reached their maximum fee too quickly, causing unnecessary overpayment to claimers.
* **Result:** No direct fund theft, but inefficient protocol spending and reduced economic efficiency.
* **Fix:** Give each prize tier its own VRGDA auction parameters (especially its own decay constant).
