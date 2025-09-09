# Redemption Drains Healthy CDPs → System-Wide Under-Collateralization

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2023-10-badger-findings/issues/199) / [One Bug Per Day](https://www.onebugperday.com/v1/583)
* **Affected Contract**: [CdpManager.sol](https://github.com/code-423n4/2023-10-badger/blob/f2f2e2cf9965a1020661d179af46cb49e993cb7e/packages/contracts/contracts/CdpManager.sol)
* **Vulnerability Type**: Business Logic / Collateral Accounting / Inconsistent Invariants

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2023-10-badger/blob/f2f2e2cf9965a1020661d179af46cb49e993cb7e/packages/contracts/contracts/CdpManager.sol#L320](https://github.com/code-423n4/2023-10-badger/blob/f2f2e2cf9965a1020661d179af46cb49e993cb7e/packages/contracts/contracts/CdpManager.sol#L320)
>
>## Vulnerability details
>
>## Impact
>
>Most actions that modify the (ICR) of a Collateralized Debt Position (CDP) must adhere to specific regulations to maintain the system's stability. For instance, closing a CDP or decreasing its collateral is not permitted if it would trigger the system's recovery mode, or if the system is already in that mode.
>
>However, the rules for redemptions are less stringent. The only conditions are that both the system and the CDPs undergoing redemption must have an TCR and ICR above the MCR. So, it is possible to redeem a cdp that is actually helping the system to stay healthy.
>
>## Proof of Concept
>
>Follow the next steps to run the coded proof of concept:
>
>* Copy and paste the following test under `test/CdpManagerTest.js`:
>
>```javascript
>it.only("redeemCollateral(): Is inconsistent with other cdp behaviors", async () => {
>    await contracts.collateral.deposit({from: A, value: dec(1000, 'ether')});
>    await contracts.collateral.approve(borrowerOperations.address, mv._1Be18BN, {from: A});
>    await contracts.collateral.deposit({from: B, value: dec(1000, 'ether')});
>    await contracts.collateral.approve(borrowerOperations.address, mv._1Be18BN, {from: B});
>    await contracts.collateral.deposit({from: C, value: dec(1000, 'ether')});
>    await contracts.collateral.approve(borrowerOperations.address, mv._1Be18BN, {from: C});
>
>    // @audit-info Current BTC / ETH price is 1.
>    const newPrice = dec(1, 18);
>    await priceFeed.setPrice(newPrice)
>
>    // @audit-info Open some CDPs.
>    await borrowerOperations.openCdp(dec(400, 18), A, A, dec(1000, 'ether'), { from: A })
>    await borrowerOperations.openCdp(dec(75, 18), B, B, dec(100, 'ether'), { from: B })
>    await borrowerOperations.openCdp(dec(75, 18), C, C, dec(100, 'ether'), { from: C })
>
>    let _aCdpId = await sortedCdps.cdpOfOwnerByIndex(A, 0);
>    let _bCdpId = await sortedCdps.cdpOfOwnerByIndex(B, 0);
>    let _cCdpId = await sortedCdps.cdpOfOwnerByIndex(C, 0);
>
>    const price = await priceFeed.getPrice()
>    
>    const currentTCR = await cdpManager.getSyncedTCR(price);
>    console.log("TCR of the system before price reduction: " + currentTCR.toString())
>
>    // @audit-info Some price reduction.
>    const newPrice2 = dec(6, 17);
>    await priceFeed.setPrice(newPrice2)
>
>    const price2 = await priceFeed.getPrice()
>
>    const currentTCR2 = await cdpManager.getSyncedTCR(price2);
>    console.log("TCR of the system after price reduction" + currentTCR2.toString())
>
>    // @audit-info All cdps are in the linked list.
>    assert.isTrue(await sortedCdps.contains(_aCdpId))
>    assert.isTrue(await sortedCdps.contains(_bCdpId))
>    assert.isTrue(await sortedCdps.contains(_cCdpId))
>
>    await cdpManager.setBaseRate(0) 
>
>    // @audit-info Redemption of 400 eBTC.
>    const EBTCRedemption = dec(400, 18)
>    await th.redeemCollateralAndGetTxObject(A, contracts, EBTCRedemption, GAS_PRICE, th._100pct)
>
>    // @audit-info Redemption will happen to A's position because B and C ICRs are below MCR.
>    assert.isFalse(await sortedCdps.contains(_aCdpId))
>    assert.isTrue(await sortedCdps.contains(_bCdpId))
>    assert.isTrue(await sortedCdps.contains(_cCdpId))
>    
>    // @audit-issue UNDERCOLLATERALIZED
>    const currentTCR3 = await cdpManager.getSyncedTCR(price2);
>    console.log("TCR after redemption: " + currentTCR3.toString())
>  })
>```
>
>This proof of concept highlights a significant risk: a situation where the entire system becomes undercollateralized. Although the likelihood of this occurring is low, there are other, more probable scenarios to consider. One such scenario involves executing one or multiple redemptions that push the system into recovery mode.
>
>Currently, there's a redemption fee in place that escalates the cost of withdrawing large amounts of collateral, serving as a financial deterrent to potential manipulation. However, this is primarily an economic obstacle. Given that eBTC aims to be a fundamental component for Bitcoin in the Ethereum DeFi ecosystem, it's crucial to ensure that redemptions do not jeopardize the system's stability.
>
>It should also be noted that the RiskDao's analysis regarding redemptions does not extend to situations where redemptions are strategically used to affect the system's collateralization.
>
>## Tools Used
>
>Manual Review
>
>## Recommended Mitigation Steps
>
>Redemptions should have the same rules for other cdp changes. For example:
>
>* Redeeming should not let the system in RM
>* Redeeming should not be possible if the system is in RM unless it makes the system to get out of RM
>
>## Assessed type
>
>Other

## Summary

The redemption mechanism allows eBTC holders to burn their stablecoins and redeem collateral from CDPs. Unlike liquidation and withdrawal flows, redemption **lacks protections that safeguard system health**.

Specifically, when some CDPs are already below MCR, redemption *skips them* and instead drains collateral from the healthiest CDPs. This can result in the **closure of strong CDPs**, leaving only under-collateralized CDPs in the system. As a result, the system's **Total Collateral Ratio (TCR)** can collapse below the required threshold, effectively destabilizing the protocol.

## A Better Explanation (With Simplified Example)

### Intended Behavior

* **Normal user actions (deposit/borrow/withdraw)**

  * Borrowing/withdrawing is only allowed if it **does not endanger the system health**.
  * Collateral ratios (ICR and TCR) are carefully enforced to avoid cascading risk.

* **Redemption (design intent)**

  * Any eBTC holder can redeem their tokens for collateral.
  * The protocol should reduce risk by redeeming against weaker CDPs, cleaning the system.

### What Actually Happens (Bug)

* Suppose 3 CDPs:

  * **A**: 1000 ETH collateral → borrows 400 eBTC (safe, above MCR).
  * **B**: 100 ETH collateral → borrows 75 eBTC (below MCR after price drop).
  * **C**: 100 ETH collateral → borrows 75 eBTC (below MCR after price drop).

* **Price drops**:

  * B and C fall below MCR.
  * A remains healthy.
  * System TCR is still > MCR, so not yet in Recovery Mode.

* **Redemption event**:

  * A user redeems 400 eBTC.
  * The system ignores B and C (already below MCR).
  * A (the safest CDP) is targeted and drained/closed.

* **Aftermath**:

  * Only B and C remain → both are unhealthy.
  * TCR drops drastically, potentially below MCR.
  * System becomes unstable.

### Why This Matters

* Normal CDP rules **prevent users from harming system health**.
* Redemption, however, **lacks those protections**.
* This allows redeemers to **strategically drain healthy CDPs** and destabilize the protocol.
* In worst cases, the system ends with **only toxic debt positions** — an undercollateralized stablecoin.

## Concrete Walkthrough (Alice, Bob, and Charlie)

* **Setup**:

  * Alice (CDP A): 1000 ETH → borrows 400 eBTC (ICR \~250%).
  * Bob (CDP B): 100 ETH → borrows 75 eBTC (ICR \~133%).
  * Charlie (CDP C): 100 ETH → borrows 75 eBTC (ICR \~133%).

* **Price drops**: ETH falls.

  * B and C are both below MCR, A remains above.

* **Mallory redeems 400 eBTC**:

  * Protocol redeems against A (since B and C are below MCR).
  * A's collateral is drained and CDP A closes.

* **Result**:

  * Only Bob and Charlie remain.
  * Both unhealthy → system TCR < MCR.
  * System is undercollateralized.

> **Analogy**: Imagine a hospital where only the strongest patients are discharged first during overcrowding. The weak patients are left inside, causing the hospital's overall health to collapse.

## Vulnerable Code Reference

⚠️ (line numbers may vary depending on the exact repo version)

1. **Redemption skips under-MCR CDPs**

    ```solidity
    // Redemption logic pseudocode
    if (ICR(cdp) < MCR) {
        // skip this CDP
        continue;
    }
    ```

2. **Targeting healthiest CDPs first**

    ```solidity
    // Selection order favors safest positions
    sortedCdps = getSortedCdpsByCollateralRatio();
    for (cdp in sortedCdps) {
        if (ICR(cdp) >= MCR) {
            redeemFrom(cdp);
        }
    }
    ```

## Recommended Mitigation

1. **Redemption should not worsen system TCR**

   * Ensure redemptions cannot close the last remaining healthy CDPs while toxic CDPs remain.

2. **Prioritize weak CDPs during redemption**

   * Instead of skipping under-MCR CDPs, redemption should **target them first** or force liquidation before redemption.

3. **System-level checks**

   * Block or limit redemptions if they push **TCR below MCR**.
   * Alternatively, allow redemptions only in **Recovery Mode** conditions.

4. **Consistency across flows**

   * Align redemption logic with borrow/withdraw invariants: no action should destabilize system health.

## Pattern Recognition Notes

* **Invariant Inconsistency**: Borrow/withdraw flows enforce system safety, but redemption bypasses those rules.
* **Skip-and-Drain Bug**: Ignoring already unhealthy CDPs causes healthy CDPs to be unfairly targeted.
* **Systemic Risk from User Arbitrage**: Attackers can deliberately redeem large amounts to destabilize TCR and profit from subsequent chaos.
* **Invariant Enforcement**: Always validate at the system level — no action (including redemptions) should reduce TCR below safe thresholds.

### Quick Recall (TL;DR)

* **Bug**: Redemption skips under-MCR CDPs and drains healthy ones first.
* **Impact**: System loses its strongest collateral, leaving only toxic CDPs → undercollateralized stablecoin.
* **Fix**: Make redemption target weak CDPs (or trigger liquidation first) and enforce system-wide TCR safeguards.
