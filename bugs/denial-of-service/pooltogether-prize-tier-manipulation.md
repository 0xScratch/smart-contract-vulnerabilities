# Prize Tier Manipulation via Single Claim Controlling `largestTierClaimed`

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2023-07-pooltogether-findings/issues/331) / [One Bug Per Day](https://www.onebugperday.com/v1/375)
* **Affected Contract**:
  * [PrizePool.sol](https://github.com/GenerationSoftware/pt-v5-prize-pool/blob/4bc8a12b857856828c018510b5500d722b79ca3a/src/PrizePool.sol#L784)
  * [Claimer.sol](https://github.com/GenerationSoftware/pt-v5-claimer/blob/57a381aef690a27c9198f4340747155a71cae753/src/Claimer.sol#L169C1-L177C4)
* **Vulnerability Type**: Denial of Service (DoS) / Incentive Manipulation / State Poisoning

## Summary

The PoolTogether prize distribution logic relies on a variable `largestTierClaimed` to determine which tiers remain active and how the maximum claim fee (`maxFee`) is computed.

Because `largestTierClaimed` is **set during prize claims**, a single malicious claimant can deliberately update it by claiming from a high tier, even at a loss. This artificially keeps that tier "active," causing liquidity to be spread thin across all tiers.

As a result, claim bots find claiming unprofitable and stop operating. Most users' prizes remain unclaimed, while the attacker's own bot continues to harvest rewards. This creates a **sustained availability attack** on automated claiming.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Close Draw**: At the end of a prize draw, the system computes liquidity distribution across tiers.
2. **Claim**: Users or bots claim prizes. The contract updates `largestTierClaimed` as claims happen, which influences whether higher tiers remain active.
3. **Healthy state**: Bots are incentivized by `maxFee` to continue claiming, keeping the system efficient and decentralized.

### What Actually Happens (Bug)

* A single claim to a high tier updates `largestTierClaimed`.
* This forces the system to treat the tier as "active," even if liquidity would otherwise make it inactive.
* Liquidity ends up too concentrated in the last tier, lowering `maxFee`.
* Because `maxFee` is too low, claim bots cannot cover gas costs profitably.
* Bots stop claiming — prizes remain unclaimed except for the attacker's own claims.

### Why This Matters

* **DoS on automated claiming** — core mechanism that ensures fair distribution is blocked.
* **Fairness loss** — attacker maintains advantage by running their own bot, while honest users are stuck.
* **Persistence** — once manipulated, state persists across draws until another manipulation resets it.

### Concrete Walkthrough (Alice & Mallory)

* **Setup**: New draw closes, liquidity distribution would normally make last tier inactive.
* **Mallory attack**:

  * Mallory claims **one prize** from the last tier, even if the gas fee > reward.
  * This updates `largestTierClaimed` to the last tier.
* **Effect on bots**:

  * Liquidity now looks thin across all tiers → computed `maxFee` is too low.
  * Third-party claim bots stop because gas > fee.
* **Result**:

  * Mallory continues to claim her own prizes manually/bot-assisted.
  * Alice (normal user) sees her prizes unclaimed, losing out.

> **Analogy**: Imagine a jackpot system where the maximum ticket a machine will check depends on the last ticket someone submitted. Mallory submits one bogus "highest number" ticket, and suddenly the machine believes the jackpot is unprofitable to process. Everyone else's tickets get ignored until Mallory lets them through.

## Recommended Mitigation

1. **Decouple tier tracking from individual claims**

   * Compute `largestTierClaimed` at draw-closing time based on aggregate statistics, not per-claim updates.

2. **Require multiple claimants / quorum**

   * Only treat a tier as "active" if a threshold of unique claimants or total claimed volume passes.

3. **Economic consistency checks**

   * Compute `maxFee` directly from liquidity math rather than depending on mutable `largestTierClaimed`.

4. **Defensive claiming logic**

   * Ignore or deprioritize claims that are obviously unprofitable and only shift global state.

## Pattern Recognition Notes

* **Single Actor State Poisoning**: When global state is updated by one actor's local action, a lone participant can bias outcomes.
* **DoS via Incentive Suppression**: By making an operation unprofitable for rational actors (bots), attackers block the system while continuing to operate at a loss for strategic gain.
* **Shared State Manipulation**: Avoid using user-driven operations to set systemic variables that affect all participants.
* **Resilience Principle**: Critical system state (like which tiers remain active) should be derived from objective, aggregate data — not mutable side-effects of claims.

### Quick Recall (TL;DR)

* **Bug**: A single claim can set `largestTierClaimed`, keeping high tiers active.
* **Impact**: Lowers `maxFee` → claim bots stop → most prizes unclaimed → attacker advantage.
* **Fix**: Derive active tiers from aggregate data at draw-close, not from single-claim updates.
