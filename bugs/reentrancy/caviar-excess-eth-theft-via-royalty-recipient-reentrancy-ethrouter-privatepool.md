# Excess ETH Theft via Royalty Recipient Reentrancy in EthRouter

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2023-04-caviar-findings/issues/569)
* **Affected Contracts**:
    - [PrivatePool.sol](https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L268)
    - [EthRouter.sol](https://github.com/code-423n4/2023-04-caviar/blob/main/src/EthRouter.sol#L140-L143)
* **Vulnerability Type**: Reentrancy / Cross-Contract State Assumption / Improper Refund Logic

## Summary

A malicious royalty recipient can exploit a cross-contract reentrancy during NFT purchases to steal users' excess ETH when trades are routed through `EthRouter`.

During `PrivatePool.buy`, royalties are paid via an external ETH transfer. If the royalty recipient is a contract, its `receive()` function executes while the router still temporarily holds user funds. The attacker can reenter `EthRouter.buy` with an empty order, triggering the router's blanket refund logic that transfers **its entire ETH balance** to the attacker.

Because the router assumes its balance belongs to the active caller, the victim's excess ETH refund is stolen.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **User calls router.buy**, sending required ETH + extra buffer.
2. Router forwards ETH to the pool.
3. Pool executes the trade and returns any unused ETH to the router.
4. Router refunds leftover ETH back to the user.

### What Actually Happens (Bug)

The execution order creates a dangerous window:

* `PrivatePool.buy` refunds excess ETH **to the router**
* Then it pays royalties via an external ETH transfer
* If the royalty recipient is malicious, it can reenter the router
* Router still holds victim ETH at this moment
* Router's refund logic sends its **entire balance** to the attacker

Result: victim's excess ETH is stolen.

### Why This Matters

* Works with standard ERC-2981 royalties
* No special permissions required
* Exploitable in a single transaction
* Particularly dangerous for batched buys
* Causes direct user fund loss

## Concrete Walkthrough (Alice & Mallory)

1. **Setup**:

    * Alice = honest buyer  
    * Mallory = NFT creator / royalty recipient (malicious contract)  
    * Router temporarily holds user ETH during execution  

2. **Step 1 — Alice buys an NFT**

    Alice sends:

    ```text
    required ETH + 10 ETH excess
    ```

    Router now temporarily holds Alice's excess.

3. **Step 2 — Router calls PrivatePool.buy**

    Inside `PrivatePool.buy`:

    1. Pool processes trade  
    2. Pool refunds unused ETH to router  
    3. Pool pays royalty to Mallory  

4. **Step 3 — Reentrancy trigger**

    When royalty is paid:

    ```solidity
    recipient.safeTransferETH(royaltyFee);
    ```

    Mallory's contract `receive()` executes.

5. **Step 4 — Mallory reenters router**

    Inside malicious `receive()`:

    ```solidity
    router.buy(new Buy, 0, false);
    ```

    Because the array is empty:

    * router skips the loop
    * immediately executes refund logic

6. **Step 5 — Router drains itself**

    Router executes:

    ```solidity
    msg.sender.safeTransferETH(address(this).balance);
    ```

    But now:

    * `msg.sender` = Mallory
    * router balance contains Alice's excess

    💥 Mallory steals the excess ETH.

7. **Step 6 — Original transaction resumes**

    Router finishes Alice's buy, but:

    * excess ETH is gone
    * Alice receives no refund

> **Analogy**: Imagine a cashier temporarily holding your change. Before giving it back, they briefly hand money to someone else, who quickly tricks the cashier into emptying the whole register to them.

## Vulnerable Code Reference

### 1) External royalty payment enables reentrancy

**PrivatePool.sol**

```solidity
// refund any excess ETH to the caller
if (msg.value > netInputAmount)
    msg.sender.safeTransferETH(msg.value - netInputAmount);

if (payRoyalties) {
    recipient.safeTransferETH(royaltyFee); // external call
}
```

⚠️ External call occurs while router still holds user funds.

### 2) Router blindly refunds entire balance

**EthRouter.sol**

```solidity
// refund any surplus ETH to the caller
if (address(this).balance > 0) {
    msg.sender.safeTransferETH(address(this).balance);
}
```

❌ Assumes all ETH belongs to the current caller
❌ Uses global balance instead of per-call accounting
❌ Reentrancy unsafe

### 3) No reentrancy protection

Neither contract uses a guard such as:

```solidity
nonReentrant
```

This allows the royalty recipient to reenter the router mid-execution.

## Recommended Mitigation

### 1. Add reentrancy protection (minimum fix)

Apply `nonReentrant` to:

* `EthRouter.buy`
* `PrivatePool.buy`
* other externally callable fund-moving functions

### 2. Avoid balance-based refunds (preferred fix)

Instead of:

```solidity
msg.sender.safeTransferETH(address(this).balance);
```

Track refunds per call:

```solidity
uint256 refund = expectedRefund;
msg.sender.safeTransferETH(refund);
```

Never rely on contract total balance inside routers.

### 3. Apply Checks-Effects-Interactions discipline

In `PrivatePool.buy`:

* minimize external calls
* consider paying royalties after router finishes
* or pull-payment model

### 4. Consider pull-based royalty payments

Safer pattern:

* accrue royalties
* recipients claim later

This removes reentrancy surface.

### 5. Add invariant tests

Recommended tests:

* router balance never decreases unexpectedly
* excess refund always returns to buyer
* malicious royalty recipient scenarios
* fuzz batched buys

## Pattern Recognition Notes

* **Cross-Contract Reentrancy Windows**: Even if a contract looks safe locally, external calls can open reentrancy into upstream routers.
* **Global Balance Refund Anti-Pattern**: Using `address(this).balance` for refunds is extremely dangerous in routers and aggregators.
* **Temporary Custody Risk**: Any router that holds user funds mid-execution must assume hostile reentry.
* **ERC-2981 Trust Assumption**: Royalty recipients are arbitrary addresses and must be treated as untrusted contracts.
* **Batch Execution Amplification**: Multi-call routers increase the likelihood of partial-state exploits.
* **Checks-Effects-Interactions Violations Across Contracts**: Security must be evaluated across call boundaries, not per contract.

## Quick Recall (TL;DR)

* **Bug**: Royalty payment enables reentrancy into router.
* **Exploit**: Attacker calls router with empty buy and drains its ETH.
* **Impact**: Users lose excess ETH refunds.
* **Root Cause**: External call + global balance refund + no reentrancy guard.
* **Fix**: Add reentrancy guard and stop using full-balance refunds.
