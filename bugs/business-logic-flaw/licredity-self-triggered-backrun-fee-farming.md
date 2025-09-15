# Self-Triggered Back-Run Enables LP Fee Farming in Licredity

* **Severity**: High
* **Source**: [Cyfrin Audit Report](https://github.com/solodit/solodit_content/blob/main/reports/Cyfrin/2025-09-01-cyfrin-licredity-v2.0.md#self-triggered-licredity_afterswap-back-run-enables-lp-fee-farming)
* **Affected Contract**: [Licredity.sol](https://github.com/Licredity/licredity-v1-core/blob/e8ae10a7d9f27529e39ca277bf56cef01a807817/src/Licredity.sol#L734-L753)
* **Vulnerability Type**: Economic Exploit / Fee Farming / Business Logic Flaw

## Summary

Licredity implements a stabilization hook `_afterSwap` that automatically executes a **back-run swap** whenever pool price goes below `1`.
The intention: prevent negative interest rates by nudging price back above `1`.

However, this **forced back-run swap pays liquidity provider (LP) fees** like any normal trade. A dominant LP can manipulate the price slightly below `1` to **self-trigger the back-run**, capturing fees from **both the attacker's push swap and the hook's automatic back-run swap**.

Because price is restored back to \~1 immediately, the attacker's capital is not at risk, and they can repeat this loop to steadily extract value from traders and the stabilization mechanism.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. User swaps base → debt.
2. If swap pushes price below 1, `_afterSwap` auto-executes a corrective swap (debt → base) to push price ≥ 1 again.
3. The back-run prevents negative interest from arising, while maintaining system stability.

### What Actually Happens (Bug)

* The back-run is a **real swap through the pool**, not a "virtual correction."
* Like all swaps, it pays LP fees.
* An attacker can LP **narrowly around price=1**, so nearly all fees in that zone flow to them.
* Attack loop:

  1. Push price slightly below 1 with base→debt swap.
  2. `_afterSwap` auto back-runs debt→base, restoring price.
  3. Withdraw LP positions and pocket fees from both swaps.
  4. Repeat with negligible price risk, since system restores price every time.

## Concrete Walkthrough (Alice & Mallory)

* **Setup**: Pool price ≈ 1.

* **Mallory** (attacker) adds 40k liquidity units in ticks **\[-2, 0]** (just below 1) and 10k units in ticks **\[0, 2]** (just above 1).

  * Below 1: captures both push and back-run swap fees.
  * Above 1: recoups small fees from price bounce back.

* **Step 1**: Mallory nudges price just above 1 with a tiny swap (ensures next push crosses downward).

* **Step 2**: Mallory performs base→debt swap (exact-out) that pushes price ≤ 1.

* **Step 3**: `_afterSwap` triggers, performing a back-run swap (debt→base), restoring price ≥ 1.

* **Step 4**: Mallory removes liquidity and collects fees from both swaps.

**Result**: Mallory's notional balance (base+debt valued 1:1) increases.
Repeating the cycle yields steady, low-risk profit.

## Vulnerable Code Reference

**1) Automatic back-run in `_afterSwap`**

```solidity
if (sqrtPriceX96 <= ONE_SQRT_PRICE_X96) {
    // back run swap to revert the effect of the current swap, using exactOut to account for fees
    IPoolManager.SwapParams memory params =
        IPoolManager.SwapParams(false, -balanceDelta.amount0(), MAX_SQRT_PRICE_X96 - 1);
    balanceDelta = poolManager.swap(poolKey, params, "");
}
```

👉 This unconditional corrective swap **routes through the AMM** and therefore **pays LP fees**.

## Recommended Mitigation

1. **Remove the back-run logic**

   * Instead of auto-swapping, simply **revert** any swap that would leave price below 1.

2. **Exclude protocol-triggered swaps from fees**

   * If a back-run must remain, do not accrue LP fees when `sender == address(this)` (the hook itself).

3. **Whitelist restrictions**

   * Only allow trusted/whitelisted addresses to LP in sub-1 ticks.

4. **Dynamic fees**

   * Increase fee rates in stabilization conditions so attackers cannot reliably profit from small nudges.

## Pattern Recognition Notes

* **Economic Loop Exploits**: Any "automatic corrective mechanism" that performs a real swap can be gamed if the attacker can position as the dominant LP.
* **Self-Triggering Hooks**: Hooks that conditionally run based on user-controlled prices may be turned into profit machines.
* **Fee Farming via Protocol Logic**: If the protocol itself pays LP fees (instead of only real traders), attackers can "farm the system."
* **Mitigation Principle**: Protocol-level corrective actions should not reward external actors — they should be cost-neutral or revert.

### Quick Recall (TL;DR)

* **Bug**: `_afterSwap` back-run executes a real swap and pays LP fees.
* **Impact**: Dominant LP around price=1 can farm fees from both legs (push + back-run), with minimal price risk.
* **Fix**: Remove/replace the back-run with reverts; or ensure back-run swaps do not accrue LP fees.
