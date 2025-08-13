# Cheap DoS via Zero-Fee TWAP Manipulation in Basin

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2023-07-basin-findings/issues/255) / [One Bug Per Day](https://www.onebugperday.com/v1/320)
* **Affected Contract**: [Well.sol](https://github.com/code-423n4/2023-07-basin/blob/c1b72d4e372a6246e0efbd57b47fb4cbb5d77062/src/Well.sol)
* **Vulnerability Type**: Economic Exploit / Denial of Service

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2023-07-basin/blob/c1b72d4e372a6246e0efbd57b47fb4cbb5d77062/src/Well.sol#L190](https://github.com/code-423n4/2023-07-basin/blob/c1b72d4e372a6246e0efbd57b47fb4cbb5d77062/src/Well.sol#L190)
>
>## Vulnerability details
>
>## Description
>
>The Well allows users to permissionless swap assets or add and remove liquidity. Users specify the intended slippage in `swapFrom`, in `minAmountOut`.
>
>The ConstantProduct2 implementation ensures `Kend - Kstart >= 0`, where `K = Reserve1 * Reserve2`, and the delta should only be due to tiny precision errors.
>
>Furthermore, the Well does not impose any fees to its users. This means that all conditions hold for a successful DOS of any swap transactions.
>
>1. Token cost of sandwiching swaps is zero (no fees) - only gas cost
>2. Price updates are instantenous through the billion dollar formula.
>3. Swap transactions along with the max slippage can be viewed in the mempool
>
>Note that such DOS attacks have serious adverse effects both on the protocol and the users. Protocol will use users due to disfunctional interactions. On the other side, users may opt to increment the max slippage in order for the TX to go through, which can be directly abused by the same MEV bots that could be performing the DOS.
>
>## Impact
>
>All swaps can be reverted at very little cost.
>
>## POC
>
>1. Evil bot sees swap TX, slippage=S
>2. Bot submits a flashbot bundle, with the following TXs
>    * Swap TX in the same direction as victim, to bump slippage above S
>    * Victim TX, which will revert
>    * Swap TX in the opposite direction and velocity to TX (1). Because of the constant product formula, all tokens will be restored to the attacker.
>
>## Tools Used
>
>Manual audit
>
>## Recommended Mitigation Steps
>
>Fees solve the problem described by making it too costly for attackers to DOS swaps. If DOS does takes place, liquidity providers are profiting a high APY to offset the inconvenience caused, and attract greater liquidity.
>
>## Assessed type
>
>DoS

## Summary

Basin's AMM design allows an attacker to cheaply disrupt user swaps by manipulating the pool's **Time-Weighted Average Price (TWAP)** right before the target transaction executes.

Because Basin charges **no swap fee**, this attack is significantly cheaper than in traditional AMMs like Uniswap. The attacker can perform a **front-run** to skew the TWAP and a **back-run** to restore the price, paying only two gas fees. This can cause targeted transactions to fail due to slippage checks, effectively **denying service** to public mempool trades.

If executed persistently, this attack can force all users to rely on private transaction relays (e.g., Flashbots), degrading UX and trust in the protocol.

## A Better Explanation (With Simplified Example)

### Intended Behavior

* Basin uses TWAP to protect trades from sudden price movements.
* A user sets a **max slippage tolerance** so their swap won't execute if the price deviates too much.
* This mechanism should prevent *unintended losses* from large trades or flash volatility.

### What Actually Happens (Vulnerability)

* Because swaps have **0% fees**, an attacker can:

  1. **Front-run**: Submit a swap right before the victim's transaction to skew the TWAP.
  2. **Back-run**: Immediately swap back after the victim's transaction is reverted, restoring the price to original levels.
* This costs **no trading fees**, only **two Ethereum gas fees** per attempt.
* Victim's transaction reverts because the manipulated TWAP breaches their slippage limit.
* The attacker gets their funds back after the back-run, losing nothing but gas.

### Example Scenario

**Pool State**: 1 BTC = 14 ETH, liquidity: 1,000 BTC & 14,000 ETH.

1. Victim sends a swap in the public mempool.
2. Attacker sees it, front-runs by swapping a large amount to move the TWAP significantly.
3. Victim's swap fails due to slippage exceeding their set tolerance.
4. Attacker back-runs, swapping back to restore original prices.

**Net result**: Attacker spends only 2 gas fees, victim swap fails, pool unaffected, attacker's funds intact.

### Why This Matters

* **Low-cost DoS**: Without fees, it's much cheaper to repeatedly execute this attack compared to other AMMs.
* **Persistent UX degradation**: If attacker sustains it, all public swaps become unreliable, forcing private mempool usage.
* **Targeted harassment possible**: Specific addresses or trades can be disrupted intentionally.

## Vulnerable Code Reference

Although the root cause is **economic** rather than a single faulty line of code, the issue arises from:

* **Swap execution logic**: No fee charged for trades (unlike Uniswap's 0.3%).
* **TWAP oracle update mechanism**: TWAP can be influenced within a single block via large trades.
* **Slippage checks**: Triggered post-TWAP manipulation, causing transaction reverts.

> This vulnerability isn't due to a coding bug in a specific function, but due to the **combination of 0% fee design + TWAP-dependent slippage checks** without front-running protection.

## Recommended Mitigation

1. **Introduce a small fee** on swaps to increase attack cost (similar to Uniswap's approach).
2. **Use last block timestamp guardrails**:

   * Ignore TWAP changes from the same block in slippage calculations.
3. **Incorporate anti-MEV protections**:

   * Encourage/require use of private mempools for critical trades.
   * Add optional transaction delay for TWAP updates before use in execution.
4. **Dynamic TWAP weighting**:

   * Reduce impact of very recent trades on the computed TWAP.

## Pattern Recognition Notes

This vulnerability is a **slippage-based Denial-of-Service (DoS) via zero-fee swaps**. To detect similar weaknesses in other AMM or DEX protocols, here are the general patterns and red flags to look for:

1. **No or Extremely Low Swap Fees**

   * Check if the protocol charges **0% fee** for swaps.
   * Zero-fee designs remove the economic cost for attackers to manipulate the pool state temporarily, making DoS or price manipulation much cheaper.
   * Even a very small fee can make these attacks uneconomical.

2. **Instantaneous State Changes with No Dampening Mechanisms**

   * AMMs where reserves (and thus prices) update instantly after any swap are highly exploitable if no fee is charged.
   * Look for formulas like the constant product `x * y = k` without time-weighted or smoothing logic.

3. **No Protection Against Mempool Transaction Ordering**

   * If swaps read `minAmountOut` from user input and execute without protection, attackers can see this in the public mempool and submit a "front-run then back-run" sandwich.
   * Red flag: No mention of `private mempool`, `Flashbots`, or on-chain anti-MEV mechanisms.

4. **Deterministic Pricing & Readable Inputs**

   * If swap outputs can be fully predicted from reserves and transaction parameters (as in most AMMs), and parameters like `minAmountOut` are public, an attacker can cheaply force reverts by pushing the price past the slippage tolerance.

5. **No Rate Limiting or Cooldown on Reserves**

   * Check if the protocol allows unlimited swaps per block without delay.
   * Without throttling, an attacker can restore the pool to its original state in the same bundle after forcing a revert, incurring no loss.

6. **MEV-Friendly Architecture**

   * Contracts with **purely permissionless swaps** and **predictable execution flow** are prime MEV bot targets.
   * If the architecture lacks any defense (commit-reveal, TWAP enforcement, randomized execution), assume attackers can reorder and bundle transactions at will.

7. **Testing Without Adversarial Simulation**

   * Lack of unit/integration tests that simulate adversarial transaction ordering is a red flag.
   * In audit reviews, explicitly simulate:

     * Swap → Victim swap (revert) → Reverse swap (restore state)
     * Measure cost — if it's just gas fees, DoS is viable.
