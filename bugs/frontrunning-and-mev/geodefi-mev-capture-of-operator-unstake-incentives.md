# MEV Capture of Operator Incentives via Public Arbitrage on Delayed Unstake

* **Severity**: Medium
* **Source**: [Consensys (Solodit)](https://solodit.cyfrin.io/issues/20754)
* **Affected Contracts**: StakeUtilsLib.sol, Swap.sol
* **Vulnerability Type**: Business Logic Flaw / MEV Exploit / Incentive Misalignment

## Summary

Geode Finance incentivizes node operators to **unstake ETH early** when the system experiences **debt (gETH de-peg)**. Withdrawn ETH is routed into the **Withdrawal Pool (DWP)**, where it is used to repay debt and restore the peg. A portion of the resulting **arbitrage surplus** is intended to reward the operator who initiated the unstake.

However, operators cannot execute this arbitrage immediately. Unstaking is a **delayed, multi-step process** that requires signaling, waiting for validator exits, and eventual finalization by an **Oracle** via `fetchUnstake()`.

Because the DWP's `swap()` function is **permissionless and externally callable**, **anyone** can capture the arbitrage once ETH liquidity becomes available. Worse, an MEV searcher can **sandwich the Oracle’s `fetchUnstake()` call**, repaying the system debt just before execution and redirecting all withdrawn ETH into surplus, which the searcher can then redeem for profit.

As a result, the **intended incentive for operators leaks entirely to MEV actors**, discouraging early unstaking and creating unhealthy conditions during severe de-pegs.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Debt appears**
   gETH trades below ETH (e.g., 1 gETH = 0.97 ETH), creating system debt.

2. **Operator action**
   A node operator signals unstake to withdraw ETH from validators.

3. **Delayed finalization**
   After hours or days, the Oracle calls `fetchUnstake()` to finalize the withdrawal.

4. **Debt repayment & reward**
   Withdrawn ETH enters the DWP, is used to repay debt, and generates arbitrage surplus.
   A portion of this surplus is paid to the operator as an incentive.

### What Actually Happens (Bug)

* Unstaking is **slow and predictable**.
* The DWP `swap()` function is **public** and **not restricted** to operators or the Oracle.
* MEV searchers monitor the mempool for `fetchUnstake()` calls.
* Because debt repayment and surplus accounting depend on **current pool state**, arbitrage can be captured **before** the operator's withdrawal is applied.

### Concrete Walkthrough (Operator vs MEV Searcher)

#### Setup

* gETH price: **0.97 ETH**
* System has **outstanding debt**
* Operator signals unstake and waits

#### Step 1: Oracle transaction appears

The Oracle submits:

```solidity
fetchUnstake(...)
```

This transaction is visible in the mempool.

#### Step 2: MEV front-run

An MEV searcher:

* Flash-borrows ETH
* Calls `swap()` in the DWP
* Buys cheap gETH
* **Repays system debt before `fetchUnstake()` executes**

Debt is now gone.

#### Step 3: `fetchUnstake()` executes

* Withdrawn ETH enters the DWP
* Since debt is already repaid:

  * **All ETH is routed to surplus**
  * Operator receives **no arbitrage bonus**

#### Step 4: MEV back-run

Searcher:

* Redeems gETH at oracle price
* Extracts ETH from surplus
* Repays flash loan
* Keeps **risk-free profit**

### Why This Matters

* Operators take **real economic and operational cost** by unstaking early.
* MEV searchers take **zero risk** and **zero capital**.
* Over time, operators rationally stop unstaking early.
* During a **severe de-peg**, the system becomes fragile:

  * Peg restoration depends on MEV behavior
  * Incentive layer fails exactly when needed most

> **End result**: The peg may recover, but the protocol loses its first-line stabilizers.

## Vulnerable Code Reference

**1) Delayed unstake finalization via Oracle**

```solidity
function fetchUnstake(
    StakePool storage self,
    DataStoreUtils.DataStore storage DATASTORE,
    uint256 poolId,
    uint256 operatorId,
    bytes[] calldata pubkeys,
    uint256[] calldata balances,
    bool[] calldata isExit
) external {
    require(
        msg.sender == self.TELESCOPE.ORACLE_POSITION,
        "StakeUtils: sender NOT ORACLE"
    );
    // ...
}
```

*Unstake completion is centralized, delayed, and predictable.*

**2) Public, permissionless DWP swap**

```solidity
function swap(
    uint8 tokenIndexFrom,
    uint8 tokenIndexTo,
    uint256 dx,
    uint256 minDy,
    uint256 deadline
)
    external
    payable
    nonReentrant
    whenNotPaused
    deadlineCheck(deadline)
    returns (uint256)
{
    return swapStorage.swap(tokenIndexFrom, tokenIndexTo, dx, minDy);
}
```

*Anyone can capture arbitrage at any time.*

## Recommended Mitigation

1. **Privileged arbitrage window**

   * Restrict DWP swaps during `fetchUnstake()` execution
   * Or give the Oracle / operator exclusive access for a short window

2. **Operator-bound reward accounting**

   * Snapshot debt at unstake-signal time
   * Guarantee operator reward regardless of intermediate swaps

3. **Delayed or averaged pricing**

   * Use TWAP-based pricing during unstake finalization to reduce sandwichability

4. **MEV-resistant execution**

   * Private mempool / commit-reveal for `fetchUnstake()`
   * Atomic debt repayment + unstake settlement

5. **Explicit incentive fallback**

   * If arbitrage is captured externally, compensate operators via protocol reserves

## Pattern Recognition Notes

* **Incentive Leakage**: When rewards depend on public, permissionless actions, MEV will capture them unless explicitly prevented.
* **Delayed Action Risk**: Any multi-step process with mempool-visible finalization is sandwichable.
* **Good Outcome ≠ Correct Incentives**: Even if the peg recovers, the system can still be economically broken.
* **MEV as Hidden Counterparty**: Always assume a faster, capital-agnostic adversary exists.
* **Stabilizer Fragility**: Systems relying on good citizen behavior must make that behavior strictly more profitable than MEV alternatives.

## Quick Recall (TL;DR)

* **Bug**: Operators unstake slowly; arbitrage is public and immediate.
* **Exploit**: MEV sandwiches `fetchUnstake()` and steals all surplus.
* **Impact**: Operators lose incentives → system weak during de-pegs.
* **Fix**: Bind rewards to operators, restrict arbitrage timing, or make unstake settlement MEV-resistant.
