# Cheap Governance Manipulation via PSM Unlimited Minting

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2023-12-ethereumcreditguild-findings/issues/335) / [One Bug Per Day](https://www.onebugperday.com/v1/781)
* **Affected Contract**:
  * [SimplePSM.sol](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/loan/SimplePSM.sol)
  * [ProfitManager.sol](https://github.com/code-423n4/2023-12-ethereumcreditguild/blob/main/src/governance/ProfitManager.sol)
* **Vulnerability Type**: Governance Manipulation / Economic Attack

## Summary

The `SimplePSM` contract allows users to deposit USDC and mint governance-enabled **gUSDC** without any **rate-limiting**. Because gUSDC carries veto power in market governance, an attacker can cheaply and temporarily acquire a large share of voting weight, exercise a veto on proposals, and immediately redeem their gUSDC back for USDC.

This creates a path for **low-cost governance griefing**: proposals can be vetoed by transient actors who have no long-term stake in the system. While no funds are directly stolen, governance can be **halted or delayed** at a fraction of the intended quorum cost.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Deposit into PSM**:

   * Alice deposits 1,000 USDC into the PSM.
   * She mints 1,000 gUSDC, a redeemable and governance-active credit token.

2. **Voting / Veto Power**:

   * Holders of gUSDC can veto governance proposals in a given market.
   * This ensures that lenders can protect themselves from harmful updates.

3. **Redeem**:

   * Alice burns her 1,000 gUSDC and withdraws 1,000 USDC back.

Governance power is supposed to come from **lenders with long-term exposure**, not temporary actors.

### What Actually Happens (Bug)

* `SimplePSM` used to rely on `RateLimitedMinter` to restrict mint throughput.
* The import remains, but rate limiting was removed to avoid issues in liquidation auctions.
* Now, **anyone can mint unlimited gUSDC instantly** by depositing USDC.

👉 Effect:

* Mallory can deposit a huge amount of USDC right before a vote, veto the proposal, then redeem immediately.
* Her governance exposure could last as little as **2 blocks**, yet she blocks legitimate system upgrades.

### Why This Matters

* The **quorum threshold** is undermined: governance power can be rented temporarily at near-zero cost.
* A single actor can repeatedly block updates, preventing protocol upgrades or safe offboarding of markets.
* While no funds are directly stolen, this creates systemic **governance paralysis**, which can erode trust and stall the protocol.

## Concrete Walkthrough (Alice & Mallory)

* **Setup**: Governance requires 1M gUSDC quorum for veto. Long-term lenders collectively hold \~1M gUSDC.

* **Mallory's attack**:

  1. Deposits 1.1M USDC into the PSM.
  2. Instantly mints 1.1M gUSDC.
  3. Uses gUSDC to veto an unfavorable governance proposal.
  4. Immediately redeems gUSDC for 1.1M USDC.

* **Result**:

  * Mallory's only cost was gas and a few blocks of exposure.
  * Governance proposal is vetoed despite long-term lenders voting otherwise.

> **Analogy**: It's like a corporate shareholder meeting where anyone can borrow shares for 2 minutes, vote down a motion, and return them without any real ownership risk.

## Vulnerable Code Reference

### 1) PSM minting without rate-limit

```solidity
// SimplePSM.sol
function deposit(uint256 amount) external {
    usdc.transferFrom(msg.sender, address(this), amount);
    gUSDC.mint(msg.sender, amount); // no rate limiting
}
```

### 2) Governance power from transient gUSDC

```solidity
// Market governance logic
if (gUSDC.balanceOf(voter) >= vetoThreshold) {
    proposal.vetoed = true;
}
```

## Recommended Mitigation

1. **Introduce a lockup period**: Require gUSDC minted via the PSM to be held for a minimum time before participating in governance.
2. **Re-introduce rate limiting (with auction-safety adjustments)**: Prevent sudden massive inflows of gUSDC that can distort governance.
3. **Separate governance vs. liquidity tokens**: Consider making PSM-minted gUSDC non-governance active, or wrapping it with a governance-only token that has lockups.
4. **Monitoring and alerts**: Track unusually large, short-lived gUSDC mints that coincide with governance events.

---

## Pattern Recognition Notes

* **Cheap Voting Rights via Transient Stake**: Systems where governance power can be gained and lost instantly are vulnerable to vote manipulation.
* **Rate Limit Trade-off**: Removing rate limits for liquidity safety (auctions) introduces governance fragility. Trade-offs should be made explicit and mitigated elsewhere.
* **Flash-Style Governance Attacks**: Even if not fully "flash-loaned," extremely short-term deposits can achieve similar effects.
* **Mitigation Pattern**: Enforce **skin in the game** via lockups, staking, or delayed redemption to align governance power with economic risk.

### Quick Recall (TL;DR)

* **Bug**: No rate limit on PSM minting → gUSDC governance power can be cheaply and temporarily acquired.
* **Impact**: Attacker vetoes proposals at negligible cost, blocking upgrades or offboarding.
* **Fix**: Add lockup, re-rate limit, or separate governance vs. liquidity exposure.
