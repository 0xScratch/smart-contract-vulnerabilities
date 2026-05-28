# Challenge Sandwich Attack via Frontrunnable Price Manipulation

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2023-04-frankencoin-findings/issues/945)
* **Affected Contract**: [`MintingHub.sol`](https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/MintingHub.sol#L140-L148)
* **Vulnerability Type**: Frontrunning / Sandwich Attack / Missing Slippage Protection / State Manipulation

## Summary

Frankencoin's challenge system allows users to challenge undercollateralized positions by depositing collateral and initiating a liquidation auction. However, `launchChallenge()` performs no validation that the challenged position's state (especially its liquidation price) remains unchanged between transaction submission and execution.

A malicious position owner can frontrun a challenger's transaction, fully repay the position debt, and reduce the position price to `0` using `adjustPrice(0)`. The challenger's transaction still executes successfully, but now the challenge operates against a position whose price is zero.

Because the challenge avert logic checks whether:

```solidity
bid >= price * collateral
```

a position with `price == 0` causes **any bid amount**, even `1 wei`, to immediately avert the challenge.

The attacker can then backrun the challenger with a near-zero bid and acquire the challenger's deposited collateral for essentially free, resulting in direct theft of challenger funds.

## A Better Explanation (With Simplified Example)

### Intended Behavior

Frankencoin uses a challenge system to liquidate unsafe positions.

1. A position owner deposits collateral and mints ZCHF.
2. If the collateral value drops too low, the position becomes unsafe.
3. A challenger may launch a challenge against the position.
4. During the challenge auction:

   * bidders compete to buy collateral,
   * unsafe debt gets resolved,
   * challenger receives a reward if liquidation succeeds.

The system assumes the position state remains economically meaningful during the challenge process.

## What Actually Happens (Bug)

The protocol allows the position owner to modify the position immediately before the challenge executes.

Specifically:

* the owner can fully repay outstanding debt,
* then reduce the liquidation price to `0`,
* while the challenger transaction is still pending in the mempool.

The challenger's transaction still succeeds because `launchChallenge()` does not verify expected state conditions.

Now the challenge exists against a position with:

* `outstanding debt = 0`
* `price = 0`

During bidding, the protocol asks whether a bid is large enough to avert the challenge:

```solidity
if (_bidAmountZCHF * ONE_DEC18 >= price * _collateralAmount)
```

Since `price == 0`, the condition becomes:

```text
bid >= 0
```

Therefore **every possible bid succeeds**, including a bid of `1 wei`.

The attacker can immediately submit a minimal bid, trigger challenge averting, and receive the challenger's collateral essentially for free.

## Why This Matters

* Challengers can lose their collateral even when they correctly identify unsafe positions.
* The attack is highly realistic because it relies on standard mempool frontrunning/sandwiching behavior.
* The victim transaction does not revert, making the attack difficult to detect in advance.
* The attacker risk is extremely low while victim losses can be significant.
* The protocol violates an important DeFi invariant:

  * users act based on observed state,
  * but execution occurs against potentially manipulated state.

## Concrete Walkthrough (Alice & Mallory)

### Initial State

Alice owns a position:

| Parameter         | Value     |
| ----------------- | --------- |
| Collateral        | 1 ETH     |
| Minted Debt       | 1500 ZCHF |
| Liquidation Price | 1500      |

Suppose ETH price falls and the position becomes undercollateralized.

Mallory notices this and decides to challenge the position.

### Step 1 — Mallory Launches Challenge

Mallory submits:

```solidity
launchChallenge(position, 1 ETH)
```

She expects:

* the unsafe position to enter liquidation,
* bidders to participate,
* and potentially earn challenger rewards.

Her transaction enters the mempool.

### Step 2 — Alice Frontruns

Alice sees the pending challenge transaction and frontruns it.

Inside a single transaction:

#### Repay Debt

```solidity
repay(1500 ZCHF)
```

Now:

```text
outstanding debt = 0
```

#### Lower Position Price

```solidity
adjustPrice(0)
```

This succeeds because the debt is already repaid.

Now:

| Parameter | Value |
| --------- | ----- |
| Debt      | 0     |
| Price     | 0     |

### Step 3 — Mallory's Challenge Executes

Mallory's challenge transaction executes normally.

The protocol:

* transfers Mallory's collateral,
* creates a challenge,
* starts the auction.

However, the challenge now references a manipulated position whose price equals zero.

Mallory is unaware of this change.

### Step 4 — Alice Backruns

Alice now submits:

```solidity
bid(challengeId, 1 wei)
```

The protocol checks:

```solidity
if (_bidAmountZCHF * ONE_DEC18 >= price * _collateralAmount)
```

Substituting values:

```text
1 wei >= 0
```

Condition passes immediately.

The challenge is considered "averted".

### Step 5 — Collateral Theft

The following code executes:

```solidity
zchf.transferFrom(msg.sender, challenge.challenger, _bidAmountZCHF);

challenge.position.collateral().transfer(
    msg.sender,
    challenge.size
);
```

Result:

| Alice Pays     | 1 wei                |
| -------------- | -------------------- |
| Alice Receives | Mallory's collateral |

Mallory loses the collateral she deposited to initiate the challenge.

> **Analogy**: Imagine participating in an auction for a damaged car. Before the auction begins, the owner secretly changes the auction rules so that *any bid automatically wins*. The owner then places a $0.01 bid and walks away with your security deposit.

## Vulnerable Code Reference

### 1) `launchChallenge()` lacks state validation

```solidity
function launchChallenge(
    address _positionAddr,
    uint256 _collateralAmount
) external validPos(_positionAddr) returns (uint256) {

    IPosition position = IPosition(_positionAddr);

    IERC20(position.collateral()).transferFrom(
        msg.sender,
        address(this),
        _collateralAmount
    );

    uint256 pos = challenges.length;

    challenges.push(
        Challenge(
            msg.sender,
            position,
            _collateralAmount,
            block.timestamp + position.challengePeriod(),
            address(0x0),
            0
        )
    );

    position.notifyChallengeStarted(_collateralAmount);
}
```

The function assumes the observed position state remains unchanged between submission and execution.

No expected price/state verification exists.

### 2) Bid avert logic depends entirely on mutable position price

```solidity
if (
    challenge.position.tryAvertChallenge(
        challenge.size,
        _bidAmountZCHF
    )
)
```

Inside the position contract:

```solidity
if (_bidAmountZCHF * ONE_DEC18 >= price * _collateralAmount)
```

The attacker manipulates `price` before the challenge executes.

If `price == 0`, every bid succeeds.

### 3) Bidder receives challenger collateral immediately

```solidity
zchf.transferFrom(
    msg.sender,
    challenge.challenger,
    _bidAmountZCHF
);

challenge.position.collateral().transfer(
    msg.sender,
    challenge.size
);
```

A minimal bid transfers the challenger's collateral to the attacker.

## Recommended Mitigation

### 1. Add Expected Price Validation (Primary Fix)

Require challengers to specify the expected position price.

Example:

```solidity
function launchChallenge(
    address position,
    uint256 collateralAmount,
    uint256 expectedPrice
) external {
    require(
        IPosition(position).price() == expectedPrice,
        "price changed"
    );
}
```

This acts similarly to slippage protection in AMMs.

If the position owner frontruns and changes the price, the challenge transaction safely reverts.

### 2. Snapshot Critical Position State

Consider snapshotting:

* price,
* debt,
* collateralization ratio,
* challenge eligibility

at challenge creation time so subsequent state manipulation cannot invalidate auction assumptions.

### 3. Restrict Dangerous Price Adjustments During Pending Challenges

Prevent:

* `adjustPrice`,
* debt repayment,
* or certain collateral modifications

while challenge initiation is pending or active.

### 4. Add Mempool-Adversarial Testing

Include tests simulating:

* frontrunning,
* backrunning,
* sandwich attacks,
* state changes between submission and execution.

## Pattern Recognition Notes

* **Missing Slippage Protection**

  * Users act based on observed state but execution occurs against mutated state.
  * Common in AMMs, lending protocols, and liquidation systems.

* **State-Dependent Economic Assumptions**

  * Auctions and liquidations are highly sensitive to mutable variables like price and debt.

* **Mempool Adversariality**

  * Public mempools allow attackers to reorder transactions and manipulate state before victim execution.

* **Sandwichable Liquidation Flows**

  * If liquidation conditions can be changed mid-flow, challengers/liquidators become attack targets.

* **Unsafe Reliance on Mutable External State**

  * Critical liquidation conditions should be snapshotted or validated atomically.

* **Economic Exploit Without Traditional Code Bug**

  * The contracts behave "as coded," but protocol assumptions fail under adversarial ordering.

## Quick Recall (TL;DR)

* **Bug**: `launchChallenge()` does not verify that position price/state remains unchanged before execution.
* **Attack**:

  1. Challenger submits challenge.
  2. Owner frontruns, repays debt, sets `price = 0`.
  3. Challenge still launches.
  4. Owner backruns with `1 wei` bid.
  5. Challenger collateral gets stolen.
* **Impact**: Direct theft of challenger collateral via sandwich attack.
* **Fix**: Add expected price/state validation (slippage protection) during challenge creation.
