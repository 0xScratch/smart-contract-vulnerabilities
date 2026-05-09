# MEV Sandwich Steals Operator Depeg Incentives During `fetchUnstake`

* **Severity**: Medium
* **Source**: [ConsenSys (Solodit)](https://solodit.cyfrin.io/issues/20754)
* **Affected Contract**: [StakeUtilsLib.sol](https://github.com/Geodefi/Portal-Eth/blob/8b07c0723d2a655a20d26620d4c3962cb9de4b00/contracts/Portal/utils/StakeUtilsLib.sol#L1276-L1288), [Swap.sol](https://github.com/Geodefi/Portal-Eth/blob/8b07c0723d2a655a20d26620d4c3962cb9de4b00/contracts/Portal/withdrawalPool/Swap.sol#L341-L358)
* **Vulnerability Type**: Incentive Failure / MEV Exploitation / Economic Design Flaw

## Summary

Geode incentivizes node operators to unstake validators during system debt or depeg situations by allowing withdrawn ETH to create profitable arbitrage opportunities in the Dynamic Withdrawal Pool (DWP). The protocol expects operators who help restore liquidity to capture this profit.

However, validator exits take time (potentially days), and the arbitrage opportunity is publicly accessible because the DWP `swap()` function is external and permissionless. As a result, MEV searchers can monitor the mempool, detect the Oracle's `fetchUnstake()` transaction, and sandwich it.

By temporarily repaying system debt before `fetchUnstake()` executes, the searcher redirects newly withdrawn ETH into protocol surplus. The searcher can then redeem discounted gETH against this surplus for profit, effectively stealing the arbitrage reward intended for node operators.

While the peg may still recover successfully, the protocol's economic incentive model breaks because operators bear the operational cost and waiting period while MEV bots capture the reward. Over time, operators may become unwilling to unstake during severe depeg events, weakening the protocol's recovery mechanism.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The protocol wants node operators to help stabilize the system during debt or depeg situations.

#### Example

Suppose:

```text
1 gETH should equal 1 ETH
```

But due to panic selling:

```text
1 gETH = 0.9 ETH
```

Now the DWP has debt and the peg is weakened.

To incentivize recovery:

1. Node operator initiates validator exit.
2. Validator unstaking process begins.
3. After several days, Oracle calls `fetchUnstake()`.
4. Withdrawn ETH enters the protocol.
5. Cheap gETH can now be redeemed closer to full ETH value.
6. Operator earns arbitrage profit for helping restore liquidity.

The protocol assumes:

```text
Operator who restores liquidity
=
Operator who earns arbitrage reward
```

This incentive is important because validator exits are slow and operationally expensive.

## What Actually Happens (Bug)

The problem is that the arbitrage opportunity is completely public.

The DWP swap system is permissionless:

```solidity
function swap(...) external
```

Meaning:

* anyone can buy discounted gETH,
* anyone can repay protocol debt,
* anyone can capture the arbitrage.

Since `fetchUnstake()` is a visible Oracle transaction, MEV searchers can exploit the timing.

## The MEV Sandwich Attack

### Initial State

* DWP currently has debt.
* gETH trades below peg.
* Operators already initiated validator exits days earlier.
* Oracle is about to call `fetchUnstake()`.

MEV bots monitor the mempool and detect:

```solidity
fetchUnstake()
```

incoming.

### Step 1 — Searcher Front-Runs

Before Oracle executes:

1. Searcher takes a flash loan in ETH.
2. Searcher buys discounted gETH from the DWP.
3. This repays the outstanding protocol debt.

At this point:

```text
Debt is now gone BEFORE fetchUnstake executes.
```

### Step 2 — Oracle Executes `fetchUnstake()`

Now the Oracle transaction runs.

Normally:

```text
withdrawn ETH → used to repay debt
```

But because the searcher already repaid debt:

```text
withdrawn ETH → goes into surplus instead
```

Fresh validator ETH now becomes redeemable liquidity.

### Step 3 — Searcher Back-Runs

The searcher now owns discounted gETH purchased earlier.

They immediately redeem it against the new surplus at near full value.

Profit:

```text
Bought cheap gETH
Redeemed for full ETH
Difference = arbitrage profit
```

Finally:

* flash loan gets repaid,
* searcher keeps the spread.

## Why This Matters

Technically:

✅ Debt gets repaired
✅ Peg improves
✅ System continues functioning

But economically:

❌ Node operator receives no reward
❌ MEV searcher captures incentive
❌ Operators lose motivation to unstake during crises

The protocol's recovery model depends on operators acting rationally and voluntarily helping restore liquidity.

If MEV consistently steals profits:

```text
Operators take operational burden
MEV bots take economic reward
```

Eventually, operators may stop participating in emergency unstaking during severe depeg scenarios.

That creates systemic risk for protocol recovery.

## Concrete Walkthrough (Operator & Searcher)

### Setup

Assume:

```text
gETH market price = 0.9 ETH
DWP currently has debt
```

Node operator starts validator exit process.

Several days later:

Oracle prepares to execute:

```solidity
fetchUnstake()
```

### Searcher Attack

#### Front-Run

Searcher sees Oracle transaction in mempool.

Searcher:

* borrows ETH via flash loan,
* buys discounted gETH,
* repays DWP debt.

Now:

```text
DWP debt = 0
```

#### Oracle Execution

`fetchUnstake()` executes.

Because debt is already repaid:

```text
withdrawn ETH enters surplus
```

instead of servicing debt.

#### Back-Run

Searcher redeems previously purchased discounted gETH using newly available surplus ETH.

Result:

```text
Searcher captures full arbitrage spread
```

Operator receives nothing despite waiting days for validator exit completion.

> **Analogy**: Imagine firefighters are promised a reward for bringing water to stop a fire. They spend days transporting water tanks to the scene. But just before they arrive, a scalper briefly sprays enough water to qualify for the reward, then immediately claims the payout using the firefighters' incoming supply. The fire still gets controlled, but the actual firefighters stop volunteering in future emergencies.

## Vulnerable Code Reference

### 1) Oracle-triggered delayed unstake finalization

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
}
```

#### Problem

Operators cannot finalize unstaking themselves.

They must:

1. wait for validator exit,
2. wait for Oracle execution,
3. hope arbitrage still exists afterward.

This creates a predictable delayed event exploitable by MEV searchers.

### 2) Public permissionless swap access

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
    virtual
    override
    nonReentrant
    whenNotPaused
    deadlineCheck(deadline)
    returns (uint256)
{
    return swapStorage.swap(tokenIndexFrom, tokenIndexTo, dx, minDy);
}
```

#### Problem

Anyone can:

* buy discounted gETH,
* repay debt,
* capture arbitrage.

No mechanism reserves recovery profits for the operators who initiated validator exits.

## Root Cause

The protocol incorrectly assumes:

```text
Liquidity provider during recovery
=
Arbitrage beneficiary
```

But in reality:

```text
Public mempool
+
Permissionless swaps
+
Delayed oracle execution
=
MEV capture opportunity
```

The incentive mechanism is therefore unenforceable.

## Recommended Mitigation

### 1. Reserve Arbitrage Rewards For Operators

Ensure operators who initiate validator exits receive exclusive or priority access to the resulting arbitrage opportunity.

Possible approaches:

* operator-specific redemption windows,
* private settlement,
* withdrawal receipts,
* dedicated reward accounting.

### 2. Reduce Oracle Predictability

Minimize publicly observable execution timing.

Possible mitigations:

* private mempool submission,
* bundled execution,
* randomized settlement timing.

This reduces MEV sandwich opportunities.

### 3. Couple Unstake Finalization With Debt Repayment Logic

Instead of allowing public users to repay debt independently before `fetchUnstake()`, the protocol could atomically:

```text
fetch unstaked ETH
+
apply debt repair
+
assign rewards
```

within a single protected flow.

### 4. Explicitly Reward Operators Separately

Instead of relying purely on market arbitrage:

* provide protocol-native incentives,
* direct rewards,
* or fixed compensation for emergency exits.

This removes dependence on publicly contestable MEV opportunities.

### 5. Add Economic Stress Testing

Protocols should test:

* adversarial MEV behavior,
* incentive compatibility,
* delayed execution assumptions,
* rational participant behavior under crisis conditions.

## Pattern Recognition Notes

* **Incentive Capture vs Intended Beneficiary**: Protocols often assume the actor performing helpful work will naturally capture the reward. In permissionless systems, MEV searchers frequently intercept those incentives.
* **Delayed Settlement Risk**: Long asynchronous processes (like validator exits) create exploitable windows between effort and reward realization.
* **Public Mempool Leakage**: Any valuable predictable state transition visible in the mempool can become MEV targetable.
* **Economic Vulnerabilities Matter**: Even without direct theft, incentive failures can destabilize protocol behavior during stress events.
* **Flash Loan Amplification**: Temporary capital access allows attackers to exploit arbitrage opportunities with near-zero upfront cost or risk.
* **Recovery Mechanism Fragility**: Crisis recovery systems should not rely on incentives that can be externally stolen.

## Quick Recall (TL;DR)

* **Bug**: MEV searchers can sandwich `fetchUnstake()` and steal arbitrage rewards intended for node operators.
* **Impact**: Operators lose incentive to unstake during depeg/debt situations, weakening protocol recovery mechanisms.
* **Root Cause**: Public swaps + predictable Oracle execution + delayed validator exits allow MEV interception.
* **Fix**: Reserve rewards for operators, reduce execution predictability, and redesign incentive accounting to resist MEV extraction.
