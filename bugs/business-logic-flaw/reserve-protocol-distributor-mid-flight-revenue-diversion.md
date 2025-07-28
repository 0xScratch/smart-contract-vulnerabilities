# Revenue Loss via Mid-Flight Distribution Parameter Changes in Reserve Protocol Distributor

- **Severity**: Medium  
- **Source**: [Code4rena](https://github.com/code-423n4/2023-06-reserve-findings/issues/34) / [One Bug Per Day](https://www.onebugperday.com/v1/256)
- **Affected Contract**: [Distributor.sol](https://github.com/reserve-protocol/protocol/blob/c4ec2473bbcb4831d62af55d275368e73e16b984/contracts/p1/Distributor.sol), [RevenueTrader.sol](https://github.com/reserve-protocol/protocol/blob/c4ec2473bbcb4831d62af55d275368e73e16b984/contracts/p1/RevenueTrader.sol), [BackingManager.sol](https://github.com/reserve-protocol/protocol/blob/c4ec2473bbcb4831d62af55d275368e73e16b984/contracts/p1/BackingManager.sol)  
- **Vulnerability Type**: Business Logic Flaw / Race Condition

## Original Bug Description

>## Lines of code
>
>[https://github.com/reserve-protocol/protocol/blob/c4ec2473bbcb4831d62af55d275368e73e16b984/contracts/p1/Distributor.sol#L61-L65](https://github.com/reserve-protocol/protocol/blob/c4ec2473bbcb4831d62af55d275368e73e16b984/contracts/p1/Distributor.sol#L61-L65)
>
>## Vulnerability details
>
>In case Distributor.setDistribution use, revenue from rToken RevenueTrader and rsr token RevenueTrader should be distributed. Otherwise wrong distribution will be used.
>
>## Proof of Concept
>
>`BackingManager.forwardRevenue` function sends revenue amount to the rsrTrader and rTokenTrader contracts, [according to the distribution inside `Distributor` contract](https://github.com/reserve-protocol/protocol/blob/c4ec2473bbcb4831d62af55d275368e73e16b984/contracts/p1/BackingManager.sol#L236-L249). For example it can 50%/50%. In case if we have 2 destinations in Distributor: strsr and furnace, that means that half of revenue will be received by strsr stakers as rewards.
>
>This distribution [can be changed](https://github.com/reserve-protocol/protocol/blob/c4ec2473bbcb4831d62af55d275368e73e16b984/contracts/p1/Distributor.sol#L61-L65) at any time.
>
>The job of `RevenueTrader` is to sell provided token for a `tokenToBuy` and then distribute it using `Distributor.distribute` function. There are 2 ways of auction that are used: dutch and gnosis. Dutch auction will call `RevenueTrader.settleTrade`, which [will initiate distribution](https://github.com/reserve-protocol/protocol/blob/c4ec2473bbcb4831d62af55d275368e73e16b984/contracts/p1/RevenueTrader.sol#L50). But Gnosis trade will not do that and user should call `distributeTokenToBuy` manually, after auction is settled.
>
>The problem that i want to discuss is next.
>
>Suppose, that governance at the beginning set distribution as 50/50 between 2 destinations: strsr and furnace. And then later `forwardRevenue` sent some tokens to the rsrTrader and rTokenTrader. Then, when trade was active to exchange some token to rsr token, `Distributor.setDistribution` was set in order to make strsr share to 0, so now everything goes to Furnace only. As result, when trade will be finished in the rsrTrader and `Distributor.distribute` will be called, then those tokens will not be sent to the strsr contract, because their share is 0 now.
>
>They will be stucked inside rsrTrader.
>
>Another problem here is that strsr holders should receive all revenue from the time, where they distribution were created. What i mean is if in time 0, rsr share was 50% and in time 10 rsr share is 10%, then `BackingManager.forwardRevenue` should be called for all tokens that has surplus, because if that will be done after changing to 10%, then strsr stakers will receive less revenue.
>
>## Tools Used
>
>VsCode
>
>## Recommended Mitigation Steps
>
>You need to think how to guarantee fair distribution to the strsr stakers, when distribution params are changed.
>
>## Assessed type
>
>Error

## Summary

The Reserve Protocol's `Distributor` contract allows governance to update distribution shares for revenue at any time. Revenue flows through `BackingManager` which forwards surplus tokens to `RevenueTrader` contracts, which execute auctions and eventually call `Distributor.distribute()` to allocate revenue to destinations. However, if governance **changes the distribution shares after surplus tokens have been forwarded but before the actual distribution**, revenue intended for `stRSR` holders can be redirected or lost, resulting in unfair rewards and potential token losses stuck inside the trader contracts.

## A Better Explanation (With Simplified Example)

### 1. How Revenue Distribution Is Intended to Work

- **BackingManager.forwardRevenue(...)** scans protocol reserves, finds surplus tokens (e.g., RSR or rToken), and forwards them to two `RevenueTrader` contracts (`rsrTrader` and `rTokenTrader`) proportionally, according to the current shares held in the `Distributor` contract at that moment.
- Each **RevenueTrader** sells unwanted tokens via auctions (Dutch or Gnosis).
- Upon auction settlement, traders call `Distributor.distribute()` to split the revenue to configured destinations (such as `FURNACE` and `ST_RSR` contracts), based on their **current distribution shares**.

### 2. The Core Issue: Mid-Flight Share Changes Break Fairness

- Suppose governance initially sets a **50-50 split** for RSR revenue between `stRSR` holders and `FURNACE`.
- BackingManager forwards 100 RSR tokens to `rsrTrader`, but `rsrTrader` cannot distribute tokens immediately—it must finish an auction.
- **Before auction settlement and actual distribution, governance changes the split** so `stRSR` gets 0% and `FURNACE` gets 100%.
- When the auction settles, `rsrTrader.distribute()` uses the **new split**, sending all revenue to `FURNACE` and zero to `stRSR`.
- The `stRSR` holders lose their rightful 50 RSR tokens, which remain stuck inside the trader contract or get burned, violating fairness and causing permanent loss.

### 3. Why This Happens

- `Distributor.setDistribution()` can be called by governance at **any time**, updating shares without regard for "in-flight" revenue batches already forwarded to RevenueTraders.
- RevenueTrader contracts call `distribute()` using the **current** shares when auction settles, not the shares effective when revenue was forwarded.
- For Gnosis auctions, `distribute()` must be called manually — creating a **wider window** for governance to change shares and divert tokens.

### 4. Example Timeline

| Time | Action                                                                    | stRSR Share | Furnace Share | RSR Tokens Held by rsrTrader | Outcome                            |
|-------|---------------------------------------------------------------------------|-------------|---------------|------------------------------|----------------------------------|
| T0    | Governance sets split: 50% stRSR, 50% Furnace                            | 50%         | 50%           | 0                            | Initial fair split set            |
| T1    | BackingManager forwards 100 RSR tokens to rsrTrader                      | 50%         | 50%           | 100                          | Equal shares owed but undistributed |
| T2    | Governance changes split: 0% stRSR, 100% Furnace                         | 0%          | 100%          | 100                          | stRSR share revoked mid-flight   |
| T3    | Gnosis auction settles; `rsrTrader.distribute()` called                  | 0%          | 100%          | 100                          | All tokens sent to Furnace; stRSR gets nothing |

### 5. Consequences

- stRSR holders lose legitimate rewards earned before governance changed shares.
- Tokens get locked or burned unintentionally, harming stakeholder trust.
- The economic guarantees of fair revenue sharing and incentive alignment are broken.

## Impact

- **Loss of Fairness and Predictability**: Revenue distributors act on **current** shares without safeguarding past entitlements, allowing governance to retroactively amend payouts.
- **Economic Harm to Stakeholders**: stRSR holders' rewards may be permanently lost or delayed, damaging protocol incentives.
- **Potential for Abuse**: Malicious or careless governance could intentionally divert funds mid-flight.
- **Undermines User Trust**: Unexpected changes to reward flows harm community confidence and participation.

## Vulnerable Code Reference

```solidity
// BackingManager.sol - Forward revenue proportionally based on Distributor totals
function forwardRevenue(IERC20[] calldata erc20s) external {
    for (...) {
        uint256 surplus = ...; // excess over safety buffer
        revenueSplit = distributor.totals(); // get current shares
        // Split surplus to rsrTrader and rTokenTrader according to shares
        safeTransfer(rsrTrader, surplus * rsrPct);
        safeTransfer(rTokenTrader, surplus * rTokenPct);
    }
}

// RevenueTrader.sol - Gnosis auction settlement requires manual distribution call
function settleTrade() external {
    // finalize auction
    // For Dutch auction, immediately call distributeTokenToBuy()
    // For Gnosis auction, distribution must be called externally 
}

// Distributor.sol - Distribution uses current shares (state) at distribution time
function distribute(IERC20 erc20, uint256 amount) external {
    uint256 totalShares = isRSR ? totals().rsrTotal : totals().rTokenTotal;
    uint256 tokensPerShare = amount / totalShares;
    for (address dest : destinations) {
        uint256 share = isRSR ? distribution[dest].rsrDist : distribution[dest].rTokenDist;
        token.safeTransferFrom(msg.sender, mapDestination(dest), tokensPerShare * share);
    }
}
```

## Recommended Mitigation

1. **Snapshot Distribution Shares at Forwarding Time**
    - When forwarding revenue from `BackingManager`, **record the distribution shares in effect for that revenue batch.**
    - `RevenueTrader` should use this snapshot during `distribute()`, ensuring payout matches the original entitlement.
2. **Lock or Delay Distribution Changes While Revenue Is In Flight**
    - Prevent governance from changing distribution splits if RevenueTrader contracts hold undistributed tokens.
    - Or implement a governance delay/timelock on distribution parameter changes to prevent abrupt mid-flight changes.
3. **Automatic and Prompt Distribution After Auctions**
    - For Gnosis auctions, automatically call `distributeTokenToBuy()` on auction settlement to minimize time windows for parameter mismatch.
4. **Enhanced Governance Transparency**
    - Emit events and require off-chain approvals for distribution parameter changes, allowing stakeholders to monitor and contest unfair changes.

## Pattern Recognition Notes

- **Mutable Critical Parameters Without Snapshots**
  - Beware of contracts allowing key parameters (e.g., revenue shares, reward rates) to be changed freely without preserving historical state. Mid-process changes can cause unfairness or lost funds.

- **Multi-Step Flows with Timing Dependencies (Race Conditions)**
  - Identify flows spanning multiple transactions (e.g., forwarding funds → auction → distribution) where state changes between steps can be exploited if intermediate data isn't locked.

- **Cross-Contract State Synchronization Issues**
  - When logic and data span multiple contracts, ensure consistent, synchronized state to prevent mismatches that cause locked assets or incorrect payouts.

- **Lack of Temporal & Access Controls on State Changes**
  - Functions modifying critical parameters should be protected by governance, timelocks, or staged workflows to prevent abrupt, unchecked changes.

- **Manual Process Steps Creating Vulnerability Windows**
  - Asynchronous or manual triggers (e.g., manual distribution call after auction) increase exposure to timing attacks or governance manipulation.

- **Placeholder vs. Real Address Handling**
  - Confirm proper substitution of placeholder addresses with real contract addresses to avoid tokens being trapped or lost.

- **Unvalidated Role Assignments & Permission Escalations**
  - Watch how privileged roles are assigned and whether they can change vital parameters without delay or restrictions, potentially enabling abuse.

- **Heuristic Summary for Auditing**
  - Find state variables controlling shares or splits.
  - Analyze how and when these are changed (immediate vs delayed).
  - Check if payout logic uses snapshots or live mutable state.
  - Identify manual steps and multi-contract dependencies.
  - Validate placeholder abstractions and access controls.
