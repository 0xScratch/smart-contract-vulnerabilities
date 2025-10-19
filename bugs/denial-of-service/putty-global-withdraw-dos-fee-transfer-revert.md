# Global Withdraw DoS via Fee Transfer Revert in PuttyV2

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2022-06-putty-findings/issues/296)
* **Affected Contract**: [PuttyV2.sol](https://github.com/code-423n4/2022-06-putty/blob/3b6b844bc39e897bd0bbb69897f2deff12dc3893/contracts/src/PuttyV2.sol)
* **Vulnerability Type**: Denial of Service (DoS) / External Call Dependency / Unchecked Fee Forwarding

## Summary

In the `withdraw()` function of **PuttyV2**, users withdraw their *strike tokens* from the contract. During this process, a small percentage fee is first transferred to the `owner()` address before the remaining tokens are sent to the user.

However, since this fee transfer is executed **before** the user's withdrawal and **not protected** against failure, any revert in the fee transfer (e.g., if the owner is the zero address or a reverting contract) causes the entire withdrawal transaction to fail.

As a result, all user withdrawals involving that base token become **permanently stuck**, leading to a **system-wide denial of service**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

When a user withdraws their strike:

1. The contract calculates a small percentage as a **fee**.
2. Sends that fee to the contract owner (`owner()`).
3. Sends the remaining amount (`strike - fee`) to the user.

```solidity
uint256 feeAmount = (order.strike * fee) / 1000;
ERC20(order.baseAsset).safeTransfer(owner(), feeAmount);
ERC20(order.baseAsset).safeTransfer(msg.sender, order.strike - feeAmount);
```

So far, it seems fine — **unless the first transfer fails**.

### What Actually Happens (Bug)

The issue is that the fee transfer happens **before** the user withdrawal, and **any revert** in that fee transfer stops the entire function.

#### Method #1 — Owner Set to Zero Address

If the owner is set to `address(0)`, most ERC20 implementations (like OpenZeppelin's) will **revert** when attempting a transfer to the zero address:

```solidity
require(to != address(0), "ERC20: transfer to the zero address");
```

Thus:

```solidity
ERC20(order.baseAsset).safeTransfer(owner(), feeAmount);
```

→ becomes

```solidity
ERC20(order.baseAsset).safeTransfer(address(0), feeAmount); // revert
```

This revert prevents the next line (the actual user transfer) from ever executing.
**Result:** Users can no longer withdraw — their strike stays stuck.

#### Method #2 — ERC777 Token as Base Asset

ERC777 tokens have a `tokensReceived()` hook that executes on every transfer.
If `owner()` points to a **malicious or faulty contract**, that contract can simply revert inside its hook whenever tokens are sent:

```solidity
function tokensReceived(...) external {
    revert("blocked");
}
```

That revert will bubble up to the PuttyV2 withdrawal, again preventing all users from withdrawing.

### Why This Matters

This is a **global DoS** — any single misconfigured or malicious owner address can block withdrawals for every user whose positions use that token as the base asset.

It also breaks the fundamental user expectation: *"I can always reclaim my funds regardless of admin behavior."*

### Concrete Walkthrough (Alice & Admin)

* **Setup**: `owner()` = Admin
* **Alice** has a position with `strike = 1000 DAI`, `fee = 1%`

#### Scenario A — Normal Flow

1. Fee = `1000 * 0.01 = 10 DAI`
2. Send 10 DAI to owner ✅
3. Send 990 DAI to Alice ✅

Alice receives her funds correctly.

#### Scenario B — Owner Set to Zero Address

1. Fee = 10 DAI
2. Send 10 DAI to `address(0)` ❌ → **revert**
3. Line never reached: user transfer fails
4. Alice's withdrawal is permanently blocked.

Since the function reverts **before** any internal state changes, **no one** can bypass this unless the contract is upgraded or redeployed.

> **Analogy**: Imagine a cashier refusing to give customers their change because the store's "tip box" is missing. Everyone's refund halts just because the tip step fails.

## Vulnerable Code Reference

**`withdraw()` function (PuttyV2.sol)**
[Lines 500-510](https://github.com/code-423n4/2022-06-putty/blob/3b6b844bc39e897bd0bbb69897f2deff12dc3893/contracts/src/PuttyV2.sol#L500)

```solidity
if ((order.isCall && isExercised) || (!order.isCall && !isExercised)) {
    uint256 feeAmount = 0;
    if (fee > 0) {
        feeAmount = (order.strike * fee) / 1000;
        ERC20(order.baseAsset).safeTransfer(owner(), feeAmount); // may revert
    }

    ERC20(order.baseAsset).safeTransfer(msg.sender, order.strike - feeAmount);
    return;
}
```

No safeguard exists to ensure that a revert in the first transfer doesn't block the rest.

## Recommended Mitigation

### ✅ 1. Use a Deferred Fee Withdrawal Pattern

Instead of transferring the fee immediately (risking revert), **record** the fee amount in a state variable and let the owner withdraw later.

```solidity
mapping(address => uint256) public ownerFees;

function withdraw(Order memory order) public {
    ...
    uint256 feeAmount = 0;
    if (fee > 0) {
        feeAmount = (order.strike * fee) / 1000;
        ownerFees[order.baseAsset] += feeAmount; // record instead of transfer
    }

    // always allow user withdrawal to succeed
    ERC20(order.baseAsset).safeTransfer(msg.sender, order.strike - feeAmount);
}

function withdrawFee(address baseAsset) public onlyOwner {
    uint256 _feeAmount = ownerFees[baseAsset];
    ownerFees[baseAsset] = 0;
    ERC20(baseAsset).safeTransfer(owner(), _feeAmount);
}
```

### ✅ 2. Fail-Safe Logic for Owner Transfers

If direct fee transfers are still desired, wrap them in a `try/catch` or a `nonRevertingTransfer()` utility that silently logs the failure but never blocks user withdrawals.

## Pattern Recognition Notes

* **External Call Dependency**: Any external call (like `safeTransfer`) can revert and should never block unrelated logic paths, especially user withdrawals.
* **DoS via Fee Forwarding**: When admin fees are paid out inline, a failed payout can globally halt all user flows.
* **Admin Configuration Risk**: Allowing `owner()` to be changed without validation (`!= address(0)`) can introduce permanent DoS vectors.
* **Withdrawal Safety Principle**: User withdrawals must never depend on third-party or admin behavior to succeed.
* **Fix Pattern**: Use *accumulate-then-withdraw* models for fee accounting to isolate failure domains.

### Quick Recall (TL;DR)

* **Bug**: Withdrawal sends owner fee before user funds. If that transfer fails, the entire withdrawal reverts.
* **Impact**: Owner can set `owner = 0x0` or a reverting contract → all user withdrawals blocked.
* **Fix**: Accumulate fees in storage and let the owner withdraw later, decoupling user flows from admin fee logic.
