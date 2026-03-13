# ERC20 Non-Zero Allowance Approval DoS in Vault Operations

* **Severity**: Medium
* **Source**: [Sherlock](https://github.com/sherlock-audit/2022-09-notional-judging/issues/59)
* **Affected Contract**: [TokenUtils.sol](https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/utils/TokenUtils.sol#L18)
* **Vulnerability Type**: ERC20 Compatibility / Denial of Service (DoS) / Unsafe Allowance Handling

## Summary

The `TokenUtils.checkApprove()` helper in **Notional Finance leveraged vaults** directly calls `approve(spender, amount)` without first resetting the allowance to `0`.

Some ERC20 tokens (notably **USDT**) enforce a safety rule where allowances **cannot be changed from a non-zero value to another non-zero value**. Attempting to do so causes the transaction to **revert**.

Because `checkApprove()` is used extensively across vault trading, liquidity management, and strategy execution logic, interactions with such tokens can cause **critical protocol operations to fail**, effectively creating a **Denial-of-Service (DoS)** for affected vault functionality.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The protocol frequently needs to allow external contracts (DEXes, Balancer vaults, wrappers) to pull tokens from the vault.

Typical workflow:

1. Vault holds tokens (e.g., USDC, WETH, etc.).
2. Vault approves an external contract to spend them.

Example:

```yaml
Vault → approve(BalancerVault, MAX_UINT)
```

Meaning:

```text
BalancerVault can transfer tokens from the vault.
```

The helper function used across the codebase:

```solidity
function checkApprove(IERC20 token, address spender, uint256 amount) internal {
    if (address(token) == address(0)) return;

    IEIP20NonStandard(address(token)).approve(spender, amount);
    _checkReturnCode();
}
```

This simply performs:

```solidity
token.approve(spender, amount)
```

### What Actually Happens (Bug)

Some ERC20 tokens (especially **USDT**) enforce the following rule:

```solidity
If currentAllowance != 0
    approve(newAmount) → revert
```

Instead, they require the allowance to be reset first:

```solidity
approve(spender, 0)
approve(spender, newAmount)
```

Since Notional **does not reset the allowance first**, the following scenario occurs:

```text
Current allowance = MAX_UINT

Protocol calls:
approve(spender, MAX_UINT)

USDT detects allowance != 0 → revert
```

Result:

```text
Transaction fails
```

### Why This Matters

`checkApprove()` is used across **many important vault flows**:

• Balancer pool interactions
• Trading utilities
• Vault strategies
• Token wrapping

If approval fails:

```text
swap fails
vault strategy fails
liquidity operations fail
```

Thus the protocol may become **unable to execute core operations involving those tokens**.

## Concrete Walkthrough (Vault & USDT Example)

### Initial State

Vault previously approved Balancer:

```solidity
allowance[vault][balancer] = MAX_UINT
```

Everything works normally.

### Later Vault Operation

The protocol calls again:

```yaml
approve(balancer, MAX_UINT)
```

Because `checkApprove()` does not reset the allowance first.

### USDT Behavior

USDT internally checks:

```solidity
if allowance != 0
    revert
```

Transaction fails.

### Result

Vault operation aborts:

```text
deposit fails
swap fails
strategy execution fails
```

Any functionality relying on this approval becomes **unusable**.

## Vulnerable Code Reference

### 1) Unsafe Approval Logic

```solidity
function checkApprove(IERC20 token, address spender, uint256 amount) internal {
    if (address(token) == address(0)) return;

    IEIP20NonStandard(address(token)).approve(spender, amount);
    _checkReturnCode();
}
```

Problem:

```text
approve() is called without resetting allowance to zero
```

### 2) Usage in Balancer Pool Utilities

```solidity
IERC20(poolContext.primaryToken)
    .checkApprove(address(Deployments.BALANCER_VAULT), type(uint256).max);

IERC20(poolContext.secondaryToken)
    .checkApprove(address(Deployments.BALANCER_VAULT), type(uint256).max);
```

These approvals are required for **liquidity operations**.

### 3) Usage in Trading Utilities

```solidity
function _approve(Trade memory trade, address spender) private {
    uint256 allowance = _isExactIn(trade) ? trade.amount : trade.limit;
    IERC20(trade.sellToken).checkApprove(spender, allowance);
}
```

Required before executing trades.

### 4) Usage in Strategy Logic

```solidity
IERC20(buyToken).checkApprove(address(Deployments.WRAPPED_STETH), amountBought);
```

Required before wrapping tokens.

## Recommended Mitigation

### 1. Reset Allowance Before Setting New Value

Implement the safe approval pattern:

```solidity
token.approve(spender, 0);
token.approve(spender, amount);
```

### 2. Use Safe Allowance Helpers

Prefer OpenZeppelin helpers:

```yaml
safeApprove
safeIncreaseAllowance
```

These properly handle edge cases.

### 3. Avoid Repeated Approvals

If using `MAX_UINT` approvals, check the current allowance first:

```solidity
if (token.allowance(address(this), spender) < amount) {
    token.approve(spender, type(uint256).max);
}
```

This avoids unnecessary approvals.

## Pattern Recognition Notes

### 1️⃣ ERC20 Compatibility Issues

Not all ERC20 tokens behave the same.

Tokens like:

```yaml
USDT
KNC (old versions)
```

require allowance reset before update.

### 2️⃣ Unsafe Approval Pattern

Red flag during audits:

```solidity
token.approve(spender, amount)
```

without:

```solidity
approve(spender, 0)
```

or safe allowance helpers.

### 3️⃣ Widely Used Helper Functions

If such logic exists inside **utility libraries**, the impact is amplified because:

```text
Many protocol features depend on them
```

### 4️⃣ Real-World Impact

This issue often results in:

```text
Protocol incompatibility
transaction reverts
core functionality failure
```

rather than direct fund theft.
