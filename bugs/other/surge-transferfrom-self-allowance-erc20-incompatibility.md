# ERC20 `transferFrom` Incompatibility via Self-Allowance Requirement

* **Severity**: Medium
* **Source**: [Sherlock](https://github.com/sherlock-audit/2023-02-surge-judging/issues/214)
* **Affected Contract**: [Pool.sol](https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L284-L293)
* **Vulnerability Type**: ERC20 Non-Standard Behavior / Compatibility Issue / Funds Stranding

## Summary

The `transferFrom` function in the Surge `Pool` contract **always deducts allowance**, even when `msg.sender == from`. This behavior **violates ERC20 standard expectations**, where allowance should **only be checked when a third party transfers tokens**.

Many DeFi protocols rely on **pull-based transfers** and use `transferFrom` even when users transfer their own tokens. Because Surge enforces allowance in all cases, these protocols **fail unexpectedly**, potentially causing **tokens to become permanently stuck**.

## A Better Explanation (With Simplified Example)

### Intended ERC20 Behavior

ERC20 standard allows two transfer mechanisms:

1. **transfer()** — user sends their own tokens
2. **transferFrom()** — third party sends tokens (requires approval)

However, **ERC20 implementations typically allow users to call `transferFrom` on themselves without needing approval**:

```solidity
if (msg.sender != from) {
    check allowance
}
```

This makes tokens **compatible with pull-based protocols**.

### What Actually Happens (Bug)

Surge's `transferFrom`:

```solidity
function transferFrom(address from, address to, uint amount) external returns (bool) {
    require(to != address(0), "Pool: to cannot be address 0");
    allowance[from][msg.sender] -= amount;
    balanceOf[from] -= amount;
    unchecked {
        balanceOf[to] += amount;
    }
    emit Transfer(from, to, amount);
    return true;
}
```

Here:

```solidity
allowance[from][msg.sender] -= amount;
```

This **always executes**, even when:

```solidity
msg.sender == from
```

This means users **must approve themselves**, which is **non-standard behavior**.

### Why This Matters

Many DeFi protocols **always use transferFrom**:

Instead of:

```solidity
token.transfer(user, amount)
```

They do:

```solidity
token.transferFrom(user, protocol, amount)
```

This normally works — but **fails with Surge tokens**.

### Concrete Walkthrough (Alice & Protocol)

#### Expected Behavior

* Alice has 100 tokens
* Alice calls:

    ```solidity
    transferFrom(Alice, Protocol, 50)
    ```

Since Alice is transferring her own tokens → should succeed

#### What Happens in Surge

* Alice calls:

    ```solidity
    transferFrom(Alice, Protocol, 50)
    ```

Surge does:

    ```solidity
    allowance[Alice][Alice] -= 50
    ```

But:

    ```solidity
    allowance[Alice][Alice] = 0
    ```

Result:

    ```text
    0 - 50 = revert
    ```

Transaction fails ❌

### Why This is Dangerous

Example scenario:

1. User deposits tokens into protocol
2. Protocol later tries to move tokens using `transferFrom`
3. Transfer fails
4. Tokens remain stuck forever

This leads to:

⚠️ **Funds becoming irreversibly stranded**

### Analogy

Imagine withdrawing money from your own bank account.

Bank says:

> "You need permission from yourself first."

That's exactly what Surge is doing.

## Vulnerable Code Reference

### Problematic `transferFrom` Implementation

```solidity
function transferFrom(address from, address to, uint amount) external returns (bool) {
    require(to != address(0), "Pool: to cannot be address 0");
    allowance[from][msg.sender] -= amount; // Always enforced (BUG)
    balanceOf[from] -= amount;
    unchecked {
        balanceOf[to] += amount;
    }
    emit Transfer(from, to, amount);
    return true;
}
```

## Recommended Mitigation

Only enforce allowance when a **third party** transfers tokens:

```solidity
function transferFrom(address from, address to, uint amount) external returns (bool) {
    require(to != address(0), "Pool: to cannot be address 0");

    if (from != msg.sender) {
        allowance[from][msg.sender] -= amount;
    }

    balanceOf[from] -= amount;

    unchecked {
        balanceOf[to] += amount;
    }

    emit Transfer(from, to, amount);
    return true;
}
```

This restores **ERC20 compatibility**.

## Pattern Recognition Notes

### 1. ERC20 Non-Standard Implementation

Red Flags:

* Custom `transferFrom`
* Missing `msg.sender != from` check
* Always deducting allowance

### 2. Pull-Based Protocol Compatibility

Protocols commonly use:

* Vaults
* Routers
* Aggregators
* Lending protocols

These rely heavily on **transferFrom**.

Breaking this pattern → breaks integrations.

### 3. Funds Stranding Risk

Compatibility bugs can cause:

* Deposits succeed
* Withdrawals fail
* Funds stuck permanently

These are **silent integration failures**.

### 4. Audit Heuristic

Always check:

* `transferFrom()` logic
* allowance deduction
* self-transfer behavior
* ERC20 standard compliance

## Quick Recall (TL;DR)

* **Bug**: `transferFrom` always deducts allowance (even for self-transfers)
* **Impact**: Breaks ERC20 compatibility
* **Risk**: Tokens can become permanently stuck in protocols
* **Fix**: Only enforce allowance when `msg.sender != from`
