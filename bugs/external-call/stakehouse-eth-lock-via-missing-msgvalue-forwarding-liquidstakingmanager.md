# ETH Locked in LiquidStakingManager Due to Missing msg.value Forwarding

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2022-11-stakehouse-findings/issues/377)
* **Affected Contract**: [LiquidStakingManager.sol](https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/liquid-staking/LiquidStakingManager.sol#L202-L215), [OwnableSmartWallet.sol](https://github.com/code-423n4/2022-11-stakehouse/blob/main/contracts/smart-wallet/OwnableSmartWallet.sol#L52-L64)
* **Vulnerability Type**: Funds Locking / Incorrect ETH Forwarding / Payable Misuse

## Summary

The `executeAsSmartWallet` function in `LiquidStakingManager` is marked as **payable**, allowing the DAO to send ETH when calling it. However, the function **does not forward `msg.value`** to the `OwnableSmartWallet.execute` call.

As a result:

* ETH sent by the DAO becomes **stuck in the `LiquidStakingManager` contract**
* Meanwhile, the `OwnableSmartWallet` sends ETH **from its own balance**
* This leads to **unexpected ETH loss** for node runners and **locked ETH** for the DAO

This creates a **funds misallocation and ETH locking vulnerability**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The DAO wants to execute a transaction **through a smart wallet** and send ETH.

Expected flow:

```yaml
DAO → LiquidStakingManager → SmartWallet → Target
```

Steps:

1. DAO calls `executeAsSmartWallet`
2. DAO sends ETH
3. Manager forwards ETH to SmartWallet
4. SmartWallet executes transaction using that ETH

### What Actually Happens (Bug)

#### The Vulnerable Function

```solidity
function executeAsSmartWallet(
    address _nodeRunner,
    address _to,
    bytes calldata _data,
    uint256 _value
) external payable onlyDAO {
    address smartWallet = smartWalletOfNodeRunner[_nodeRunner];
    require(smartWallet != address(0), "No wallet found");

    IOwnableSmartWallet(smartWallet).execute(
        _to,
        _data,
        _value
    );
}
```

The function is **payable**, but:

❌ `msg.value` is **never forwarded**
❌ ETH stays in `LiquidStakingManager`

## Why This Is Dangerous

Inside `OwnableSmartWallet`:

```solidity
return target.functionCallWithValue(callData, value);
```

This sends ETH **from SmartWallet balance**, not from DAO.

So:

| Step                 | Result           |
| -------------------- | ---------------- |
| DAO sends ETH        | Goes to manager  |
| Manager calls wallet | No ETH forwarded |
| Wallet executes      | Uses its own ETH |

This causes:

* DAO ETH locked in manager
* Node runner loses ETH from wallet

## Concrete Walkthrough (Alice & DAO)

### Setup

```text
Manager balance = 0 ETH
SmartWallet balance = 4 ETH
```

DAO calls:

```yaml
executeAsSmartWallet{value: 1.5 ETH}
```

Expected:

```text
DAO → 1.5 ETH → SmartWallet → target
```

Actual:

```text
DAO → 1.5 ETH → Manager (locked)
SmartWallet → 1.5 ETH → target
```

Final State:

```text
Manager = 1.5 ETH (locked)
SmartWallet = 2.5 ETH
```

### Result

* DAO loses 1.5 ETH (locked)
* Node runner loses 1.5 ETH (unexpected deduction)

## Why It Doesn't Revert

The smart wallet already has enough ETH:

```text
SmartWallet balance = 4 ETH
Required = 1.5 ETH
```

So transaction succeeds and the bug remains **silent**.

This makes the vulnerability **harder to detect**.

## Vulnerable Code Reference

### 1) Payable Function Without Forwarding ETH

```solidity
function executeAsSmartWallet(...) external payable onlyDAO {
    ...
    IOwnableSmartWallet(smartWallet).execute(
        _to,
        _data,
        _value
    );
}
```

Missing:

```solidity
{value: msg.value}
```

### 2) Smart Wallet Sends From Its Own Balance

```solidity
function execute(
    address target,
    bytes memory callData,
    uint256 value
)
external
payable
onlyOwner
returns (bytes memory)
{
    return target.functionCallWithValue(callData, value);
}
```

This uses wallet balance instead of forwarded ETH.

## Recommended Mitigation

### Fix: Forward msg.value

```solidity
IOwnableSmartWallet(smartWallet).execute{value: msg.value}(
    _to,
    _data,
    _value
);
```

This ensures:

```yaml
DAO → Manager → SmartWallet → Target
```

## Additional Defensive Improvements

### 1. Validate Value Consistency

```solidity
require(msg.value == _value, "Value mismatch");
```

Prevents incorrect usage.

### 2. Emit Event for Visibility

```solidity
emit SmartWalletExecution(_nodeRunner, _value);
```

Helps track fund movement.

### 3. Avoid Payable If Not Needed

If ETH not required:

```yaml
Remove payable modifier
```

## Pattern Recognition Notes

### Payable Function Without Forwarding

If:

```text
function payable → external call → no {value: msg.value}
```

⚠️ Possible ETH lock vulnerability

### Middle Contract ETH Sink

Architecture:

```yaml
User → Manager → Wallet
```

Always verify:

* ETH forwarded?
* ETH stuck?
* value mismatch?

### Silent Fund Loss

When:

* Wallet has enough balance
* Transaction succeeds
* Funds misallocated

These bugs are **high risk and hard to detect**

## Quick Recall (TL;DR)

* **Bug**: `msg.value` not forwarded to smart wallet
* **Impact**: DAO ETH locked + node runner ETH lost
* **Cause**: Payable function missing `{value: msg.value}`
* **Fix**: Forward ETH when calling smart wallet
