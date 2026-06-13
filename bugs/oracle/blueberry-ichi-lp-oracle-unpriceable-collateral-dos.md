# ICHI LP Tokens Cannot Be Priced, Causing Global Position Opening DoS

* **Severity**: Medium
* **Source**: [Sherlock](https://github.com/sherlock-audit/2023-02-blueberry-judging/issues/152)
* **Affected Contract**: [IchiLpOracle.sol](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/IchiLpOracle.sol#L19-L39)
* **Vulnerability Type**: Denial of Service (DoS) / Oracle Dependency Failure / Unsupported Asset Pricing

## Summary

Blueberry allows users to open leveraged positions using ICHI LP tokens as collateral. Before a position can be created, the protocol must determine the value of the deposited LP tokens.

The `IchiLpOracle` uses a Fair LP Pricing model that calculates LP token value from the prices of the underlying assets held by the vault. This requires reliable oracle prices for both underlying tokens.

However, one of the underlying assets is often the `ICHI` token itself. Since neither Chainlink nor Band provides a price feed for ICHI, the oracle cannot determine the LP token value. As a result, every attempt to open a new position using `IchiVaultSpell` reverts during collateral valuation, making the entire ICHI integration unusable.

## A Better Explanation (With Simplified Example)

### Intended Behavior

When a user opens a leveraged position:

1. User deposits ICHI LP tokens as collateral.
2. Blueberry calculates the value of those LP tokens.
3. The protocol determines:

   * Total collateral value.
   * Borrow value.
   * Loan-to-value ratio (LTV).
   * Liquidation thresholds.
4. If the position is safe, it is created successfully.

The LP valuation is performed using a Fair LP Pricing formula:

```text
LP Value =
(Value of Token0 Reserves + Value of Token1 Reserves)
/
LP Supply
```

To calculate reserve values, the oracle must know the price of both underlying assets.

### What Actually Happens (Bug)

Suppose an ICHI vault contains:

```text
ICHI
USDC
```

The oracle attempts:

```text
Price(ICHI)
Price(USDC)
```

Obtaining the USDC price is easy because oracle feeds exist.

However:

```text
Price(ICHI)
```

fails because neither Chainlink nor Band supports ICHI.

Without the ICHI price:

```text
ICHI Reserve Value = Unknown
```

which means:

```text
Total LP Value = Unknown
```

which means:

```text
Collateral Value = Unknown
```

which means:

```text
Position Safety Check Cannot Execute
```

Therefore the transaction reverts.

### Why This Matters

The protocol fundamentally relies on collateral valuation.

If collateral cannot be priced:

* Position health cannot be calculated.
* Liquidation thresholds cannot be calculated.
* Borrow limits cannot be calculated.
* Risk checks cannot be performed.

As a result, every position creation attempt fails.

Unlike a temporary oracle outage, this is a permanent integration failure because the required oracle feed simply does not exist.

### Concrete Walkthrough (Alice Example)

#### Setup

An ICHI vault contains:

```text
100 ICHI
500 USDC
```

Assume:

```text
USDC = $1
```

but:

```text
ICHI = Unknown
```

because no oracle supports it.

#### Alice Opens a Position

Alice deposits ICHI LP tokens as collateral.

Blueberry attempts to calculate LP value:

```text
LP Value =
(Value of ICHI Reserve + Value of USDC Reserve)
/
LP Supply
```

To compute this, the oracle requests:

```text
Price(ICHI)
```

#### Oracle Failure

The base oracle routes to Chainlink/Band:

```text
getPrice(ICHI)
```

No supported feed exists.

The call fails.

#### Position Creation Failure

Since LP value cannot be determined:

```text
Collateral Value = Unknown
```

The protocol cannot execute its final safety checks:

```text
Is Position Safe?
Can User Borrow This Amount?
Would Position Immediately Be Liquidatable?
```

Because these calculations require collateral value, the transaction reverts.

#### Result

```text
Alice → Revert
Bob → Revert
Charlie → Revert
Everyone → Revert
```

Every new position using the ICHI integration becomes impossible to create.

> **Analogy:** Imagine a bank accepts gold bars as collateral but has no way to determine the market price of gold. Since the bank cannot determine the collateral value, it cannot safely issue loans against it. Every loan application using gold collateral would be rejected, regardless of how much gold the customer owns.

## Vulnerable Code Reference

### 1) LP valuation depends on underlying asset prices

```solidity
(uint256 r0, uint256 r1) = vault.getTotalAmounts();

uint256 px0 = base.getPrice(address(token0));
uint256 px1 = base.getPrice(address(token1));
```

The oracle requires valid prices for both underlying assets.

---

### 2) LP value calculation cannot proceed without those prices

```solidity
uint256 totalReserve =
    (r0 * px0) / 10**t0Decimal +
    (r1 * px1) / 10**t1Decimal;

return (totalReserve * 1e18) / totalSupply;
```

If either `px0` or `px1` cannot be obtained, LP valuation fails.

---

### 3) Unsupported ICHI price feed causes failure

```solidity
uint256 px0 = base.getPrice(address(token0));
```

When `token0` (or `token1`) is ICHI:

```text
getPrice(ICHI)
```

cannot be resolved through Chainlink or Band.

The oracle reverts, preventing collateral valuation.

## Recommended Mitigation

### 1. Add Alternative Oracle Support For ICHI (Primary Fix)

Implement an alternative pricing source for ICHI when Chainlink/Band feeds do not exist.

Possible sources:

* Uniswap TWAP.
* SushiSwap TWAP.
* Other reputable AMM-based TWAP systems.

Example conceptually:

```solidity
if (chainlinkFeedExists(token)) {
    return chainlinkPrice(token);
}

return twapPrice(token);
```

### 2. Validate Asset Oracle Availability During Integration

Before approving a collateral type, ensure all underlying assets can be priced.

Example:

```text
Can token0 be priced?
Can token1 be priced?
```

Only enable vaults that satisfy both requirements.

### 3. Add Deployment-Time Validation

During protocol configuration:

```text
New LP Vault Added
    ↓
Verify Oracle Support For All Components
    ↓
Enable Vault
```

This prevents unusable vaults from being listed.

### 4. Add Integration Tests

Create tests that verify:

* Every supported LP asset can be priced.
* Every underlying token has a valid oracle source.
* Position creation succeeds for every approved vault.

## Pattern Recognition Notes

* **Oracle Dependency Risk**: A protocol may correctly implement pricing logic but still fail if required oracle feeds do not exist.
* **Collateral Must Be Priceable**: Any asset accepted as collateral must have a reliable valuation mechanism. Unsupported assets effectively become unusable collateral.
* **Integration Assumption Failure**: Developers assumed all underlying assets would have oracle support. The protocol never validated this assumption.
* **Indirect Asset Dependencies**: Even if the protocol only accepts LP tokens, it still depends on the price availability of every underlying asset inside those LP tokens.
* **Deployment Validation Matters**: Many integration bugs arise because external assumptions are never verified during onboarding.

## Quick Recall (TL;DR)

* **Bug**: ICHI LP pricing requires an ICHI oracle price, but Chainlink and Band do not support ICHI.
* **Root Cause**: The LP oracle assumes all underlying assets have available price feeds.
* **Impact**: LP valuation fails, collateral value cannot be calculated, and all new ICHI-backed positions revert.
* **Result**: The entire `IchiVaultSpell` integration becomes unusable.
* **Fix**: Add an alternative oracle source (such as TWAP) and validate oracle availability before onboarding collateral assets.
