# Phantom Surplus Settlement via Omitted Portfolio Margin Borrow Liabilities

* **Severity**: Medium
* **Source**: [Code4rena](https://code4rena.com/audits/2026-04-monetrix/submissions/F-141)
* **Affected Contracts**:

  * [PrecompileReader.sol](https://github.com/code-423n4/2026-04-monetrix/blob/main/src/core/PrecompileReader.sol)
  * [MonetrixAccountant.sol](https://github.com/code-423n4/2026-04-monetrix/blob/main/src/core/MonetrixAccountant.sol)
  * [MonetrixVault.sol](https://github.com/code-423n4/2026-04-monetrix/blob/main/src/core/MonetrixVault.sol)
* **Vulnerability Type**: Accounting Error / Liability Omission / Phantom Yield Distribution

## Summary

Monetrix calculates protocol backing by reading Portfolio Margin (PM) supplied balances from Hyperliquid's PM state. However, the PM state contains both **supplied assets** and **borrow liabilities**, while Monetrix only reads and accounts for the supplied side.

As a result, Portfolio Margin debt is completely ignored during backing calculations.

This causes the protocol to overestimate its net assets and report a positive surplus even when the PM account is actually underwater after considering borrow obligations. Since settlement logic relies on this surplus calculation, the protocol can distribute yield that was never truly earned.

The result is the creation of **phantom yield**: newly minted USDM and real USDC distributions backed by accounting errors rather than actual economic gains.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The protocol periodically determines whether it has generated profit.

Conceptually:

```text
Backing = Assets - Liabilities

Surplus = Backing - USDM Supply
```

If surplus is positive:

```text
Protocol made money
→ Yield can be distributed
```

If surplus is negative:

```text
Protocol is underwater
→ Yield should not be distributed
```

The entire settlement process relies on backing being calculated correctly.

### What Actually Happens (Bug)

Hyperliquid's Portfolio Margin state contains both:

```text
Supply = Assets owned by the account
Borrow = Debt owed by the account
```

Example:

```text
Supply = 100,000 USDC
Borrow = 80,000 USDC
```

Real PM equity:

```text
100,000 - 80,000
=
20,000 USDC
```

However, Monetrix only reads:

```text
Supply = 100,000
```

and completely ignores:

```text
Borrow = 80,000
```

Therefore Monetrix believes:

```text
Backing Contribution = 100,000
```

instead of:

```text
Backing Contribution = 20,000
```

This inflates backing and causes surplus calculations to become incorrect.

### Why This Matters

The protocol uses surplus to determine whether yield may be distributed.

Suppose:

```text
PM Supply = 100,000
PM Borrow = 80,000
USDM Supply = 50,000
```

#### Reality

```text
Backing = 20,000

Surplus
=
20,000 - 50,000
=
-30,000
```

The protocol is underwater.

Yield distribution should be impossible.

#### Monetrix's View

Borrow debt is ignored:

```text
Backing = 100,000

Surplus
=
100,000 - 50,000
=
50,000
```

Now the protocol incorrectly believes it generated:

```text
50,000 profit
```

even though it is actually:

```text
30,000 underwater
```

This false profit is called **phantom surplus**.

### How Phantom Surplus Becomes Real Money

Settlement logic trusts the inflated surplus value.

If settlement approves:

```text
2,000 USDM yield
```

the protocol proceeds to:

1. Mint new USDM.
2. Inject that yield into sUSDM.
3. Pay protocol allocations (insurance/foundation).
4. Allow users to later redeem the value.

The accounting error therefore becomes actual value extraction from protocol reserves.

The protocol is effectively paying rewards from capital that does not exist.

### Concrete Walkthrough (Alice & Mallory)

#### Setup

Vault PM position:

```text
Supply = 1,000,000 USDC
Borrow = 1,000,000 USDC
```

Real PM equity:

```text
0
```

Additionally:

```text
10,000 USDC
```

of real profit exists elsewhere in the vault.

Total USDM Supply:

```text
1,990,000
```

Mallory deposits and receives sUSDM shares before settlement.

#### What Should Happen

Real backing:

```text
10,000
```

Real surplus:

```text
10,000 - 1,990,000
=
negative
```

No yield should be distributed.

#### What Monetrix Sees

Borrow debt is omitted:

```text
PM Supply = +1,000,000
PM Borrow = ignored
```

Resulting backing appears significantly larger than reality.

The accountant now reports positive distributable surplus.

#### Settlement Executes

Settlement passes validation and distributes yield.

New USDM is minted and injected into sUSDM.

Since Mallory already owns sUSDM shares, she receives part of the phantom yield.

#### Extraction

Mallory:

1. Waits through cooldown.
2. Claims the increased USDM balance.
3. Redeems excess USDM for USDC.

Real protocol reserves leave the system even though the underlying profit never existed.

> **Analogy**: Imagine a company calculating profit using only assets on its balance sheet while completely ignoring loans. The company believes it is profitable, pays bonuses to shareholders, and slowly depletes its remaining cash despite never actually making money.

## Vulnerable Code Reference

### 1) PM Reader Decodes Only Supply Value

```solidity
function suppliedBalance(address account, uint64 token)
    internal
    view
    returns (uint64 supplied)
{
    (bool ok, bytes memory res) =
        HyperCoreConstants.PRECOMPILE_SUPPLIED_BALANCE.staticcall(
            abi.encode(account, token)
        );

    require(ok && res.length >= 128);

    (,,, supplied) = abi.decode(
        res,
        (uint64, uint64, uint64, uint64)
    );
}
```

The PM state contains:

```text
(
 borrowBasis,
 borrowValue,
 supplyBasis,
 supplyValue
)
```

Only `supplyValue` is decoded.

Borrow information is discarded.

### 2) Accountant Credits PM Supply As Backing

```solidity
total += int256(
    PrecompileReader.suppliedUsdcEvm(account)
);
```

and

```solidity
total += int256(
    PrecompileReader.suppliedNotionalUsdcFromPerp(...)
);
```

Both paths add supplied value to backing.

No corresponding liability subtraction exists.

### 3) Inflated Backing Feeds Surplus Calculation

```solidity
function surplus() public view returns (int256) {
    return totalBackingSigned()
        - int256(usdm.totalSupply());
}
```

Since backing is inflated:

```text
Backing too high
→ Surplus too high
→ Distributable surplus too high
```

### 4) Settlement Trusts Inflated Surplus

```solidity
int256 ds = distributableSurplus();

require(ds > 0);
require(proposedYield <= uint256(ds));
```

The settlement gate trusts the faulty accounting.

No verification of PM liabilities occurs.

### 5) Phantom Yield Is Materialized

```solidity
usdm.mint(address(this), userShare);

susdm.injectYield(userShare);
```

The accounting mistake becomes newly created yield for sUSDM holders.

## Recommended Mitigation

### 1. Account For PM Borrow Liabilities

The reader should decode all PM fields:

```solidity
(
    borrowBasis,
    borrowValue,
    supplyBasis,
    supplyValue
)
```

instead of only reading `supplyValue`.

### 2. Use Net PM Equity

Backing should be computed as:

```text
Net PM Value
=
Supplied Assets
-
Borrow Liabilities
```

rather than considering supplied assets alone.

### 3. Prefer Native Net-Equity Reads

If Hyperliquid exposes a direct account-equity field, use it instead of reconstructing equity from individual supply positions.

Direct equity values reduce the risk of future accounting mismatches.

### 4. Add Accounting Invariant Tests

Tests should verify:

```text
Assets = Supply - Borrow
```

and ensure:

```text
Settlement cannot execute
when net surplus is negative.
```

## Pattern Recognition Notes

* **Assets Without Liabilities**: Whenever a protocol counts assets, verify that associated debts are also included. Missing liabilities often create phantom profits.

* **Accounting Reconstruction Risk**: Rebuilding account equity from partial external state is dangerous. If possible, consume authoritative net-equity values instead.

* **Profit Calculation Trust Boundary**: Settlement systems frequently trust accounting modules. A bug in accounting often propagates directly into reward distribution.

* **Phantom Yield Pattern**: If a protocol calculates yield from accounting surplus, any overstatement of surplus can lead to minting rewards that are not economically backed.

* **Debt-Blind Integrations**: Integrations with lending, margin, or leverage systems should always be reviewed for asset-side accounting that forgets liability-side accounting.

## Quick Recall (TL;DR)

* **Bug**: Monetrix counts Portfolio Margin supplied assets but ignores Portfolio Margin borrow liabilities.
* **Impact**: Backing becomes overstated, surplus becomes inflated, and settlement can distribute yield that does not actually exist.
* **Exploitation**: Users holding sUSDM during settlement receive a share of phantom yield and can later redeem it for real value.
* **Root Cause**: The PM reader decodes only `supplyValue` and discards `borrowValue`.
* **Fix**: Include PM borrow liabilities in backing calculations or use a direct net-equity value from Hyperliquid.
