# Global Withdraw DoS via Zero-Amount Queue Poisoning in VUSD

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2022-02-hubble-findings/issues/119)
* **Affected Contract**: [VUSD.sol](https://github.com/code-423n4/2022-02-hubble/blob/main/contracts/VUSD.sol#L53)
* **Vulnerability Type**: Denial of Service (DoS) / Input Validation / Queue Poisoning

## Summary

`VUSD.sol` implements a **queue-based withdrawal system** where users burn vUSD and are added to a FIFO queue for later USDC redemption. The `processWithdrawals()` function processes withdrawals in batches, constrained by `maxWithdrawalProcesses`.

Because the `withdraw()` function **does not enforce a minimum withdrawal amount**, an attacker can enqueue **arbitrarily many zero-amount withdrawals**. These entries consume processing slots in the queue but transfer no value.

Since withdrawals must be processed **strictly in order** and only a limited number can be processed per transaction, an attacker can flood the queue with zero-value entries. This forces the system (or governance/users) to spend **prohibitively high gas** to clear the queue before legitimate withdrawals can be reached, resulting in a **practical global DoS** of withdrawals.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Deposit (mint)**:

   * User deposits USDC.
   * Contract mints equivalent vUSD (1:1 backing).

2. **Withdraw**:

   * User burns vUSD.
   * A withdrawal request is appended to the queue:

     ```solidity
     withdrawals.push(Withdrawal(msg.sender, amount));
     ```

3. **Processing**:

   * Anyone can call `processWithdrawals()`.
   * Processes up to `maxWithdrawalProcesses` entries per call.
   * Transfers USDC to users and advances the `start` pointer.

### What Actually Happens (Bug)

* `withdraw()` allows:

  ```solidity
  withdraw(0)
  ```
* This:

  * Burns nothing (`burn(0)` is valid)
  * Still appends a queue entry

👉 Result: attacker can create **millions of zero-value entries**

### Why This Matters

* Queue is processed **sequentially (FIFO)**
* Processing is **gas-limited per call**
* Zero withdrawals still:

  * Consume loop iterations
  * Cost gas to process

👉 So attacker can:

* Spam queue cheaply (only gas)
* Force system to spend **massive cumulative gas** to recover

### Concrete Walkthrough (Alice & Mallory)

* **Setup**:

  ```text
  start = 0
  withdrawals = []
  ```

* **Mallory attack**:

  * Calls `withdraw(0)` repeatedly (e.g., 1,000,000 times)
  * Queue becomes:

  ```text
  [0, 0, 0, 0, ..., 0, (Alice: 100 USDC)]
  ```

* **Alice withdraws**:

  * Alice submits legitimate withdrawal (100 USDC)
  * Her request is **at the end of the queue**

* **Processing begins**:

  * Each `processWithdrawals()` call handles ~100 entries

* **System behavior**:

  ```text
  Call 1 → process 100 zeros  
  Call 2 → process next 100 zeros  
  ...  
  Call 10,000 → finally reach Alice  
  ```

### 💣 Impact

* To reach real withdrawals:

  * Requires **thousands of transactions**
  * Each costs gas

👉 If no one is willing to pay:

> withdrawals effectively **never get processed**

> **Analogy**:
> A checkout line where only 100 customers can be served per day.
> An attacker inserts 1 million fake customers with empty carts.
> Real customers must wait months unless someone pays to process the fake ones.

## Vulnerable Code Reference

**1) Missing minimum amount validation in `withdraw`**

```solidity
function withdraw(uint amount) external {
    burn(amount); // allows amount = 0
    withdrawals.push(Withdrawal(msg.sender, amount)); // poison entry
}
```

**2) Gas-limited batch processing**

```solidity
while (i < withdrawals.length && (i - start) <= maxWithdrawalProcesses)
```

* Only a bounded number of entries processed per call

**3) Strict FIFO processing**

```solidity
Withdrawal memory withdrawal = withdrawals[i];
```

* Cannot skip entries
* Must process all prior entries before reaching legitimate ones

## Recommended Mitigation

### 1. Enforce minimum withdrawal amount (primary fix)

```solidity
function withdraw(uint amount) external {
    require(amount > 0, "amount must be > 0");
    burn(amount);
    withdrawals.push(Withdrawal(msg.sender, amount));
}
```

Or stronger (economic deterrence):

```solidity
require(amount >= 1e6); // 1 USDC (6 decimals)
```

### 2. Skip zero-value entries during processing (defensive)

```solidity
if (withdrawal.amount == 0) {
    i++;
    continue;
}
```

### 3. Consider non-FIFO or skip-capable design

* Allow skipping malformed entries
* Prevent single class of entries from blocking progress

### 4. Introduce economic friction

* Withdrawal fee
* Queue insertion cost

### 5. Alternative design (strongest)

* Replace global queue with **user-claim model**

  * Users withdraw independently
  * No shared queue bottleneck

## Pattern Recognition Notes

### 🔹 Queue Poisoning via Cheap Entries

If:

* Users can insert entries cheaply
* Processing is bounded

👉 Attackers can flood the system and shift cost to others

### 🔹 Gas Asymmetry Attack

* Attacker cost: **O(N)** (one-time)
* Victim cost: **O(N)** (repeated across many txs)

👉 Leads to **economic DoS**

### 🔹 FIFO + No Skipping = Fragility

Strict ordering means:

* Bad entries must be processed first
* No escape hatch → system slowdown

### 🔹 Missing Input Validation

* Allowing `amount = 0` created:

  * Free state pollution
  * Infinite spam vector

### 🔹 Unbounded Data Structures

* Dynamic arrays with no limits
* No pruning mechanism

👉 Classic audit red flag

## Quick Recall (TL;DR)

* **Bug**: `withdraw(0)` allowed → attacker spams zero-value queue entries
* **Impact**: Queue becomes massive → requires huge gas to clear → **withdrawals effectively frozen**
* **Fix**: Enforce `amount > 0` (or minimum threshold), optionally skip invalid entries and rethink queue design
