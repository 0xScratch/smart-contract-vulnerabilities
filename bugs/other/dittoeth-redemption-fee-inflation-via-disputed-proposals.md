# Redemption Fee Inflation via Disputed Redemption Proposals

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2024-03-dittoeth-findings/issues/274)
* **Affected Contract**: [RedemptionFacet.sol](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/RedemptionFacet.sol#L204)
* **Vulnerability Type**: Economic Manipulation / Incorrect State Accounting / Peg Stability Failure

## Summary

`RedemptionFacet` implements DittoETH's redemption mechanism, where users can redeem dUSD for ETH collateral from short positions. To prevent excessive redemptions, the protocol maintains a dynamic `baseRate` that increases after redemptions and makes future redemptions more expensive.

However, the protocol updates the redemption `baseRate` **when a redemption is proposed**, not when the redemption is successfully finalized.

Because proposed redemptions can later be disputed and reverted, an attacker can create invalid redemption proposals that increase the global redemption fee, then dispute their own proposals from another account. The redemption itself is cancelled, but the increased `baseRate` remains.

This allows an attacker to artificially inflate redemption fees without actually reducing dUSD supply, making legitimate redemptions unprofitable and preventing dUSD from restoring its peg.

## A Better Explanation (With Simplified Example)

### Intended Behavior

When dUSD trades below $1, redemptions should restore the peg.

Example:

* dUSD price drops to `$0.95`.
* Alice buys `1000 dUSD` from the market for `$950`.
* Alice redeems through DittoETH.
* She receives approximately `$1000` worth of ETH collateral.

Profit:

```
$1000 ETH received
-
$950 purchase cost

= $50 profit
```

This arbitrage:

1. Creates buying pressure for dUSD.
2. Removes dUSD supply.
3. Pushes dUSD back toward `$1`.

To prevent too many redemptions, DittoETH increases redemption fees after large redemption activity:

```
Large redemption
        ↓
baseRate increases
        ↓
future redemption fee increases
```

Over time, the fee decays back down.

### What Actually Happens (Bug)

The problem is the timing of the fee update.

Current flow:

```
proposeRedemption()
        |
        v
calculateRedemptionFee()
        |
        v
baseRate increased immediately

        |
        v

wait dispute period

        |
        |
        +----> disputed
                 |
                 v
          redemption cancelled
          BUT baseRate stays increased
```

The protocol treats a proposed redemption as if it already happened.

But a proposal is only a candidate and can still fail.

Therefore:

```
Actual redeemed dUSD = 0

but

Protocol thinks redemption happened
```

The global redemption fee becomes artificially inflated.

---

### Why This Matters

The redemption system is DittoETH's main peg recovery mechanism.

If dUSD falls:

```
dUSD = $0.95
```

normally:

```
Buy cheap dUSD
        ↓
Redeem for ETH
        ↓
Profit
        ↓
Peg restored
```

But after the attack:

```
dUSD = $0.95

Expected redemption profit:
5%

Artificial redemption fee:
10%

Result:
Redeeming loses money
```

No rational user redeems.

The peg recovery mechanism stops working.

## Concrete Walkthrough (Alice Attacker)

### Initial State

```
dUSD price = $0.95

Asset.baseRate = 0%

dUSD supply = 1,000,000
```

The protocol needs redemptions to happen.

---

### Step 1: Alice Creates Bad Redemption Proposal

Alice intentionally submits an incorrect redemption slate.

Correct order should be:

```
ShortRecord A:
CR = 120%

ShortRecord B:
CR = 130%

ShortRecord C:
CR = 140%
```

Lowest collateral ratios should be redeemed first.

Instead Alice proposes:

```
Redeem:

ShortRecord C
CR = 140%
```

This proposal is invalid because weaker positions exist.

---

### Step 2: Protocol Updates Fee Immediately

During `proposeRedemption()`:

```solidity
uint88 redemptionFee =
    calculateRedemptionFee(
        asset,
        p.totalColRedeemed,
        p.totalAmountProposed
    );
```

Inside:

```solidity
uint256 newBaseRate =
    decayedBaseRate + redeemedDUSDFraction;

Asset.baseRate = uint64(newBaseRate);
```

The protocol updates:

```
baseRate:

0%
 |
 v
5%
```

even though nothing has actually been redeemed yet.

---

### Step 3: Alice Disputes Herself

Using another wallet:

```
Wallet A:
creates invalid proposal

Wallet B:
disputes proposal
```

The dispute succeeds.

The protocol restores:

```
ShortRecord debt     ✓
ShortRecord collateral ✓
```

Example:

```solidity
currentSR.collateral += currentProposal.colRedeemed;

currentSR.ercDebt += currentProposal.ercDebtRedeemed;
```

The redemption is undone.

---

### Step 4: Fee State Is Not Restored

The missing rollback:

```
Asset.baseRate
Asset.lastRedemptionTime
```

These remain modified.

Result:

```
Actual redemption:

0 dUSD

Fee impact:

baseRate increased
```

The attacker created fake redemption volume.

---

### Why The Penalty Does Not Stop This

The dispute system assumes different actors.

Normally:

```
Alice creates bad proposal

Bob disputes

Alice loses penalty
Bob earns penalty
```

But attackers can use multiple accounts:

```
Wallet A loses penalty

Wallet B receives penalty
```

Net loss:

```
≈ 0
```

The protocol cannot know both wallets have the same owner.

---

## Undercollateralized ShortRecord Amplification

The attack becomes even cheaper when an undercollateralized ShortRecord exists.

Relevant logic:

```solidity
p.colRedeemed = p.oraclePrice.mulU88(
    p.amountProposed
);

if (p.colRedeemed > currentSR.collateral) {
    p.colRedeemed = currentSR.collateral;
}
```

Example ShortRecord:

```
Debt:
10,000 dUSD

Collateral:
$10 ETH
```

Redeeming should return:

```
$10,000 ETH
```

but only `$10` exists.

Therefore:

```
colRedeemed = $10
```

The fee paid uses:

```solidity
redemptionRate * colRedeemed
```

So the attacker pays fees on:

```
$10
```

However, the baseRate increase uses:

```solidity
ercDebtRedeemed
```

Meaning:

```
Fee increase based on:

10,000 dUSD
```

The attacker gets:

```
Huge baseRate increase
+
Tiny actual cost
```

---

## Vulnerable Code Reference

### 1) Redemption fee updated during proposal stage

```solidity
function proposeRedemption(...)
{
    ...

    uint88 redemptionFee =
        calculateRedemptionFee(
            asset,
            p.totalColRedeemed,
            p.totalAmountProposed
        );

    ...
}
```

The proposal is not final yet.

It can still be disputed.

---

### 2) `calculateRedemptionFee()` permanently updates global state

```solidity
uint256 newBaseRate =
    decayedBaseRate + redeemedDUSDFraction;

Asset.baseRate = uint64(newBaseRate);

Asset.lastRedemptionTime = protocolTime;
```

This assumes the redemption definitely happened.

---

### 3) Dispute restores SR state but not fee state

```solidity
currentSR.collateral += currentProposal.colRedeemed;

currentSR.ercDebt += currentProposal.ercDebtRedeemed;
```

Restored:

```
✓ collateral
✓ debt
```

Missing:

```
✗ Asset.baseRate
✗ Asset.lastRedemptionTime
```

---

## Recommended Mitigation

### 1. Apply Redemption Fee Only After Successful Redemption

Move fee state updates from:

```solidity
proposeRedemption()
```

to:

```solidity
claimRedemption()
```

New flow:

```
propose redemption

        ↓

dispute period

        ↓

successful claim

        ↓

increase baseRate
```

Only real redemptions affect future fees.

---

### 2. Roll Back Fee Changes During Successful Disputes

Alternative approach:

Store the previous fee state:

```solidity
previousBaseRate
previousRedemptionTime
```

If dispute succeeds:

```solidity
Asset.baseRate = previousBaseRate;

Asset.lastRedemptionTime = previousRedemptionTime;
```

However, this is more complex because fee decay depends on time.

---

### 3. Separate Pending and Finalized Accounting

Track:

```
pending redemption volume
```

separately from:

```
completed redemption volume
```

Only finalized redemptions should affect economic parameters.

## Pattern Recognition Notes

* **Premature State Updates**: Do not permanently update global economic variables before an operation becomes final.

* **Rollback Incompleteness**: If an action can be reverted through a dispute/challenge system, every state change caused by that action must also be reversible.

* **Economic Accounting ≠ Execution Attempt**: Attempted volume should not affect parameters designed around completed volume.

* **Sybil-Assisted Incentive Bypass**: Never rely on penalties between addresses when one actor can control both sides.

* **Delayed Finality Systems**: In propose → challenge → execute flows, irreversible effects should happen only in the execute phase.

## Quick Recall (TL;DR)

* **Bug**: `baseRate` increases during `proposeRedemption()` before redemption is final.
* **Attack**: Submit invalid redemption → increase fees → dispute yourself → redemption cancelled but fees remain high.
* **Impact**: Artificially high redemption fees make dUSD arbitrage unprofitable, breaking peg recovery.
* **Fix**: Update redemption fee only after successful `claimRedemption()` or fully rollback fee state during disputes.
