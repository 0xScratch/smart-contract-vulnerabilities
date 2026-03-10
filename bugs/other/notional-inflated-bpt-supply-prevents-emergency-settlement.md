# Emergency Settlement Bypass via Inflated BPT Supply in Balancer Phantom Pool

* **Severity**: High
* **Source**: [Sherlock](https://github.com/sherlock-audit/2022-12-notional-judging/issues/11)
* **Affected Contract**: [Boosted3TokenAuraVault.sol](https://github.com/sherlock-audit/2022-12-notional/blob/main/contracts/vaults/Boosted3TokenAuraVault.sol), [SettlementUtils.sol](https://github.com/sherlock-audit/2022-12-notional/blob/main/contracts/vaults/balancer/internal/settlement/SettlementUtils.sol#L86)
* **Vulnerability Type**: Logic Error / Incorrect Accounting / Risk-Control Bypass

## Summary

The vault uses **Balancer Boosted Pools**, which rely on **Phantom BPT tokens**. In these pools, `totalSupply()` does **not represent the circulating supply** of BPT because most tokens are pre-minted and held internally by the pool.

However, the function `getEmergencySettlementBPTAmount()` incorrectly calculates the pool size using:

```solidity
IERC20(pool).totalSupply()
```

instead of the correct **virtual supply**.

Because Phantom pools often have extremely large `totalSupply` values (e.g., ~`2^111`), the computed **pool share threshold becomes massively inflated**.

This causes the emergency settlement condition to **never trigger**, even when the vault actually owns an excessive share of the liquidity pool.

As a result, monitoring systems and keepers relying on this function may incorrectly assume the vault is healthy and **fail to trigger emergency settlement**, potentially leaving the vault with an unsafe concentration of pool liquidity.

## A Better Explanation (With Simplified Example)

### Key Concept: Phantom BPT Supply

In standard liquidity pools:

```yaml
totalSupply == circulating supply
```

But in **Balancer Phantom pools**:

```yaml
totalSupply ≠ circulating supply
```

Instead:

```yaml
virtualSupply = circulating BPT actually owned by LPs
```

Why?

At pool creation, Balancer **pre-mints a massive amount of BPT** and keeps most of it internally for accounting mechanics.

Example:

```yaml
totalSupply = 2^111  (~2.5e33)
actual LP supply = 100 BPT
```

So the real pool share calculations must use:

```yaml
virtualSupply
```

### Intended Behavior

The vault enforces a safety rule:

```yaml
vault BPT holdings ≤ maxBalancerPoolShare * pool supply
```

This ensures the vault does **not control too much liquidity** in the pool.

If the vault exceeds the threshold:

```yaml
Emergency Settlement is triggered
```

This forces the vault to exit its Balancer position.

### What Actually Happens (Bug)

The code calculates pool supply incorrectly:

```solidity
totalBPTSupply = IERC20(pool).totalSupply();
```

Since Phantom pools return extremely large numbers, the computed threshold becomes enormous.

Example:

```yaml
totalSupply = 2^111
maxBalancerPoolShare = 5%
```

Threshold becomes:

```yaml
~1.2e32 BPT
```

Meanwhile the vault may only hold:

```yaml
vaultBPTHeld = 200 BPT
```

The safety check:

```solidity
if (vaultBPTHeld <= threshold)
    revert InvalidEmergencySettlement
```

Now becomes:

```yaml
200 ≤ 1e32  → always true
```

So the function **always reverts**, preventing emergency settlement.

### Why This Matters

The function is designed for **keepers and monitoring bots** to determine whether emergency settlement should occur.

But due to this bug:

* The function **always reverts**
* Off-chain systems interpret this as **"no emergency settlement needed"**
* Vault may accumulate an unsafe portion of pool liquidity.

This creates a risk scenario where the vault holds **too large of a pool share**, making exits difficult or highly slippage-prone.

## Concrete Walkthrough (Vault & Pool Scenario)

### Setup

Balancer Boosted Pool:

```yaml
totalSupply = 2^111
virtualSupply = 100 BPT
maxBalancerPoolShare = 20%
```

Real threshold should be:

```yaml
20% of 100 = 20 BPT
```

### Actual Calculation (Bug)

Protocol calculates:

```yaml
20% of 2^111 ≈ enormous threshold
```

### Vault Position

Vault owns:

```yaml
vaultBPTHeld = 40 BPT
```

Real situation:

```yaml
40 / 100 = 40% pool share
```

This **exceeds the allowed 20%** and should trigger emergency settlement.

But the contract sees:

```yaml
40 ≤ enormous threshold
```

Thus:

```solidity
revert InvalidEmergencySettlement
```

Emergency settlement **never triggers**.

### Analogy

Imagine a pizza restaurant that prints **1,000,000 ownership coupons**, but only **100 are actually distributed**.

The system checks:

```yaml
Does someone own more than 10%?
```

If it uses:

```yaml
10% of 1,000,000 = 100,000 coupons
```

But only 100 coupons exist in circulation.

So **nobody ever exceeds the threshold**, even if someone owns **50 out of 100**.

That's exactly what happens here.

## Vulnerable Code Reference

### 1) Incorrect BPT Supply Source

```solidity
function getEmergencySettlementBPTAmount(uint256 maturity) 
    external 
    view 
    returns (uint256 bptToSettle)
{
    Boosted3TokenAuraStrategyContext memory context = _strategyContext();

    bptToSettle = context.baseStrategy._getEmergencySettlementParams({
        maturity: maturity, 
        totalBPTSupply: IERC20(context.poolContext.basePool.basePool.pool).totalSupply()
    });
}
```

Problem:

```yaml
totalSupply() used instead of virtualSupply
```

### 2) Threshold Calculation

```solidity
uint256 emergencyBPTWithdrawThreshold =
    settings._bptThreshold(totalBPTSupply);
```

## 3) Safety Check That Always Reverts

```solidity
if (strategyContext.vaultState.totalBPTHeld <= emergencyBPTWithdrawThreshold)
    revert Errors.InvalidEmergencySettlement();
```

Due to inflated `totalBPTSupply`, this condition is **always true**.

## Recommended Mitigation

### 1) Use Virtual Supply Instead of Total Supply

Replace:

```solidity
totalBPTSupply = IERC20(pool).totalSupply();
```

With:

```solidity
totalBPTSupply = context.poolContext._getVirtualSupply(context.oracleContext);
```

This ensures the pool share calculation uses the **actual circulating BPT supply**.

### 2) Add Defensive Validation

Consider verifying that supply values are within reasonable ranges:

```solidity
require(totalBPTSupply > 0)
```

This prevents unexpected accounting failures.

### 3) Add Monitoring Tests

Include tests that ensure emergency settlement triggers correctly when:

```solidity
vaultBPTHeld > maxBalancerPoolShare * virtualSupply
```

## Pattern Recognition Notes

### Phantom Supply vs Real Supply

Protocols integrating with Balancer boosted pools must distinguish between:

```solidity
totalSupply (internal accounting)
virtualSupply (circulating LP supply)
```

Using the wrong metric breaks share calculations.

### Risk-Control Logic Depends on Correct Accounting

Any logic enforcing limits such as:

* maximum pool share
* collateralization
* exposure caps

is **only as reliable as the supply metric used**.

Incorrect accounting silently disables safety checks.

### External Protocol Assumptions

When integrating with external protocols (Balancer, Curve, Uniswap):

* their **token accounting mechanisms may differ**
* assumptions like `totalSupply == circulating supply` may be false.

Always verify protocol-specific supply definitions.

### Off-Chain Keeper Reliance

Many DeFi systems rely on:

```yaml
view functions → bot decisions
```

If a view function **returns incorrect information or reverts unexpectedly**, keepers may **fail to trigger safety mechanisms**.

## Quick Recall (TL;DR)

* **Bug**: `totalSupply()` used instead of `virtualSupply()` for Phantom BPT pools.
* **Effect**: Pool supply becomes massively inflated (~`2^111`).
* **Impact**: Emergency settlement threshold becomes unreachable.
* **Result**: Vault may hold an unsafe share of pool liquidity and cannot be emergency settled in time.
* **Fix**: Use **Balancer virtual supply** when calculating pool share limits.
