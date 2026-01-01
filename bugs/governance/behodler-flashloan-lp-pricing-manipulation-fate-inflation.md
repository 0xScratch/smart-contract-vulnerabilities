# Flashloan Manipulation of LP Pricing in burnAsset & setEYEBasedAssetStake

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2022-01-behodler-findings/issues/304)
* **Affected Contract**: [`LimboDAO.sol`](https://github.com/code-423n4/2022-01-behodler/blob/main/contracts/DAO/LimboDAO.sol)
* **Vulnerability Type**: Price Oracle Manipulation / Flashloan Attack / Governance Power Inflation

## Summary

The `burnAsset` function (and similarly `setEYEBasedAssetStake`) in `LimboDAO` calculates the amount of **Fate** (governance voting power) granted when burning (or staking) EYE-based LP tokens by relying on the **current spot reserves** of the underlying Uniswap/Sushiswap pool.

An attacker can **temporarily inflate** the perceived EYE value inside the LP using a flashloan + large swap. This makes each burned/staked LP token appear to contain much more EYE than it actually does → **attacker receives significantly more Fate** than intended, at the cost of only swap fees.

With extra Fate, the attacker gains outsized influence over LimboDAO governance (proposals, parameter changes, token onboarding) or — if enabled — can convert it into Flan tokens for profit.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Burn LP tokens** → contract reads current pool reserves to compute how much **EYE** your LP share represents.
2. Fate awarded = implied EYE amount × 20 (for LPs) → quadratic staking is slow, burning is fast & linear.
3. Normal user gets fair Fate based on real economic value contributed.

### What Actually Happens (Bug)

* The calculation trusts **instant pool state**:

  ```solidity
  uint256 actualEyeBalance = IERC20(domainConfig.eye).balanceOf(asset);  // ← manipulable in same tx
  uint256 eyePerUnit = (actualEyeBalance * ONE) / totalSupply;
  uint256 impliedEye = (eyePerUnit * amount) / ONE;
  ```

* Flashloan allows arbitrary temporary skew of `actualEyeBalance` (by dumping EYE into the pool → price of EYE pumps → LP looks EYE-rich).
* After manipulation → burn LP → get inflated Fate → reverse manipulation & repay flashloan.

### Why This Matters

* **Governance takeover risk**: Large Fate lets attacker dominate votes, lodge strong proposals, or block legitimate changes.
* **Profit via Flan**: If `fateToFlan` rate > 0, attacker can convert fake Fate into real Flan tokens.
* **Low cost**: Only swap fees (~0.3%) + gas; no permanent capital lockup.
* **Same issue** exists in staking function (`setEYEBasedAssetStake`).
* Classic pattern — similar to **Cheese Bank** (~$3M drained) & **Warp Finance** (~$8M) exploits, where LP collateral was overvalued via flashloan.

### Concrete Walkthrough (Mallory Attack)

* **Setup**: EYE-LINK pool = 1000 EYE + 1000 LINK, total LP supply = 1000. Mallory owns 100 LP (10% share).
* **Normal burn** → Fate ≈ 1000 × (100/1000) × 20 = **2000 Fate**
* **Attack** (single tx via flashloan):
  1. Flashloan 1000+ EYE
  2. Swap all into pool for LINK → pool now ≈ 2000 EYE + 500 LINK
  3. Burn 100 LP → contract sees "your share now has 2000 × 10% = 200 EYE" → Fate ≈ **4000** (double!)
  4. Swap back 500 LINK → recover original 1000 EYE
  5. Repay flashloan + fee
* **Result**: Double (or more) Fate for ~swap fee cost.

> **Analogy**: Like using a flashloan to temporarily pump a stock price, then selling your shares at the inflated price, and reversing everything — but here the "sale" is burning LP for governance power points.

## Vulnerable Code Reference

**1) Manipulable spot price read in `burnAsset`**

```solidity
uint256 actualEyeBalance = IERC20(domainConfig.eye).balanceOf(asset);   // ← flashloan can change this
uint256 eyePerUnit = (actualEyeBalance * ONE) / totalSupply;
uint256 impliedEye = (eyePerUnit * amount) / ONE;
fateCreated = impliedEye * 20;   // ← inflated!
```

**2) Same issue in `setEYEBasedAssetStake`**

```solidity
uint256 actualEyeBalance = IERC20(domainConfig.eye).balanceOf(asset);   // ← same vulnerable read
// ... then impliedEye check & fateWeight = 2 * rootEYE (still based on manipulable value)
```

**Lines referenced in original report**: ~L356 & ~L392 (2022 contest version)

## Recommended Mitigation

1. **Primary fix** — Use manipulation-resistant LP pricing (e.g. Alpha Finance fair LP formula or TWAP)
   * Compute fair reserves using **Uniswap V2/V3 TWAP** (time-weighted average price) + constant product invariant
   * Avoid single-block spot reserves entirely
2. **Alternative** — Use external oracle (Chainlink) for EYE price, then derive LP value safely
3. **Additional hardening**:
   * Add max % change limits on implied EYE per burn/stake
   * Consider capping Fate from burning in low-liquidity pools
   * Require minimum pool liquidity threshold for approved assets
4. **Tests & invariants**:
   * Forge/Hardhat tests simulating flashloan + manipulation → assert Fate not inflated
   * Fuzz testing with extreme pool skews

## Pattern Recognition Notes

* **Spot Price Oracle Risk**: Relying on current pool reserves without time-weighting is one of the most common flashloan vectors in DeFi (2020-2022 era).
* **Governance-as-Collateral**: When governance power (Fate) is minted based on manipulable values, it becomes "free" influence.
* **Burning as Fast Power**: Linear/exponential multipliers (×20 here) amplify manipulation rewards.
* **No Funds Drained ≠ Low Risk**: Even without direct theft, governance capture can lead to protocol changes that harm users long-term.
* **Historical Lessons**: Cheese Bank, Warp Finance, bZx — all fixed with TWAP/fair pricing; modern protocols avoid spot reserves for valuation.

## Quick Recall (TL;DR)

* **Bug**: `burnAsset` & `setEYEBasedAssetStake` use manipulable current pool reserves to value EYE in LP → flashloan inflates → extra Fate minted cheaply.
* **Impact**: Outsized governance power (vote domination, proposal spam) or Flan minting profit.
* **Fix**: Switch to TWAP-based or fair LP pricing (e.g. Alpha formula); avoid spot reserves for critical calculations.
* **Status (from 2022 report)**: Confirmed, but downgraded to Medium (no direct fund loss).
